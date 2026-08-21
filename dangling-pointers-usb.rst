===============================================================================
Use-After-Free and potential Double-Free in UVC Binding and Unbinding Functions
===============================================================================

Introduction
============

What are these binding and unbinding functions?
-----------------------------------------------
These functions(will be mentioned in the passage below) are responsible
for binding and unbinding uvc gadget devices. We will focus on the failure
path of the binding function and standard unbinding function.

What is a Use-After-Free?
-------------------------
A Use-After-Free occurs when a piece of data(could be a struct, an int or
anything) is freed, but another function attempts to use it. It could be a
result of races or dangling pointers(Dangling pointers are pointers that
point to data that has been freed, but itself is not set to ``NULL``)

What is a Double-Free?
----------------------
A Double-Free is an occurence where data that has been freed is attempted
to be freed again. This freed space may be allocated to another process after
the first free and thus causes problems. It may occur due to incorrect refcounting.

The Problem
===========

In our binding function in UVC Gadget driver, ``uvc_function_bind()`` to be exact,
we branch off to a failure path(fallback) if the following conditions are not
satisfied:
1. ``if (!ep)`` where ``ep = usb_ep_autoconfig(cdev->gadget, &uvc_interrupt_ep);``
2. or second case of ``!ep`` where ``ep = usb_ep_autoconfig(cdev->gadget, &uvc_hs_streaming_ep);``
3. ``if (uvc->control_req == NULL || uvc->control_buf == NULL)``
4. ``if (v4l2_device_register(&cdev->gadget->dev, &uvc->v4l2_dev))``

Here is the full function:

.. code-block:: c
   static int
	uvc_function_bind(struct usb_configuration *c, struct usb_function *f)
	{
		struct usb_composite_dev *cdev = c->cdev;
		struct uvc_device *uvc = to_uvc(f);
		struct uvcg_extension *xu;
		struct usb_string *us;
		unsigned int max_packet_mult;
		unsigned int max_packet_size;
		struct usb_ep *ep;
		struct f_uvc_opts *opts;
		int ret = -EINVAL;
		uvcg_info(f, "%s()\n", __func__);
		scoped_guard(mutex, &uvc->lock)
			uvc->func_unbound = false;
		opts = fi_to_f_uvc_opts(f->fi);
		/* Sanity check the streaming endpoint module parameters. */
		opts->streaming_interval = clamp(opts->streaming_interval, 1U, 16U);
		opts->streaming_maxpacket = clamp(opts->streaming_maxpacket, 1U, 3072U);
		opts->streaming_maxburst = min(opts->streaming_maxburst, 15U);
		/* For SS, wMaxPacketSize has to be 1024 if bMaxBurst is not 0 */
		if (opts->streaming_maxburst &&
		    (opts->streaming_maxpacket % 1024) != 0) {
			opts->streaming_maxpacket = roundup(opts->streaming_maxpacket, 1024);
			uvcg_info(f, "overriding streaming_maxpacket to %d\n",
				  opts->streaming_maxpacket);
		}
		/*
		 * Fill in the FS/HS/SS Video Streaming specific descriptors from the
		 * module parameters.
		 *
		 * NOTE: We assume that the user knows what they are doing and won't
		 * give parameters that their UDC doesn't support.
		 */
		if (opts->streaming_maxpacket <= 1024) {
			max_packet_mult = 1;
			max_packet_size = opts->streaming_maxpacket;
		} else if (opts->streaming_maxpacket <= 2048) {
			max_packet_mult = 2;
			max_packet_size = opts->streaming_maxpacket / 2;
		} else {
			max_packet_mult = 3;
			max_packet_size = opts->streaming_maxpacket / 3;
		}
		uvc_fs_streaming_ep.wMaxPacketSize =
			cpu_to_le16(min(opts->streaming_maxpacket, 1023U));
		uvc_fs_streaming_ep.bInterval = opts->streaming_interval;
		uvc_hs_streaming_ep.wMaxPacketSize =
			cpu_to_le16(max_packet_size | ((max_packet_mult - 1) << 11));
		/* A high-bandwidth endpoint must specify a bInterval value of 1 */
		if (max_packet_mult > 1)
			uvc_hs_streaming_ep.bInterval = 1;
		else
			uvc_hs_streaming_ep.bInterval = opts->streaming_interval;
		uvc_ss_streaming_ep.wMaxPacketSize = cpu_to_le16(max_packet_size);
		uvc_ss_streaming_ep.bInterval = opts->streaming_interval;
		uvc_ss_streaming_comp.bmAttributes = max_packet_mult - 1;
		uvc_ss_streaming_comp.bMaxBurst = opts->streaming_maxburst;
		uvc_ss_streaming_comp.wBytesPerInterval =
			cpu_to_le16(max_packet_size * max_packet_mult *
				    (opts->streaming_maxburst + 1));
		/* Allocate endpoints. */
		if (opts->enable_interrupt_ep) {
			ep = usb_ep_autoconfig(cdev->gadget, &uvc_interrupt_ep);
			if (!ep) {
				uvcg_info(f, "Unable to allocate interrupt EP\n");
				goto error;
			}
			uvc->interrupt_ep = ep;
			uvc_control_intf.bNumEndpoints = 1;
		}
		uvc->enable_interrupt_ep = opts->enable_interrupt_ep;
		/*
		 * gadget_is_{super|dual}speed() API check UDC controller capitblity. It should pass down
		 * highest speed endpoint descriptor to UDC controller. So UDC controller driver can reserve
		 * enough resource at check_config(), especially mult and maxburst. So UDC driver (such as
		 * cdns3) can know need at least (mult + 1) * (maxburst + 1) * wMaxPacketSize internal
		 * memory for this uvc functions. This is the only straightforward method to resolve the UDC
		 * resource allocation issue in the current gadget framework.
		 */
		if (gadget_is_superspeed(c->cdev->gadget))
			ep = usb_ep_autoconfig_ss(cdev->gadget, &uvc_ss_streaming_ep,
						  &uvc_ss_streaming_comp);
		else if (gadget_is_dualspeed(cdev->gadget))
			ep = usb_ep_autoconfig(cdev->gadget, &uvc_hs_streaming_ep);
		else
			ep = usb_ep_autoconfig(cdev->gadget, &uvc_fs_streaming_ep);
		if (!ep) {
			uvcg_info(f, "Unable to allocate streaming EP\n");
			goto error;
		}
		uvc->video.ep = ep;
		uvc_fs_streaming_ep.bEndpointAddress = uvc->video.ep->address;
		uvc_hs_streaming_ep.bEndpointAddress = uvc->video.ep->address;
		uvc_ss_streaming_ep.bEndpointAddress = uvc->video.ep->address;
		/*
		 * Hold opts->lock across both the XU string-descriptor fixup below and
		 * the descriptor-copy block further down.  Without this, configfs
		 * uvcg_extension_drop() (which takes opts->lock) can race with the
		 * list_for_each_entry() walks here and inside uvc_copy_descriptors(),
		 * leading to a UAF on a freed struct uvcg_extension.  See
		 * drivers/usb/gadget/function/uvc_configfs.c::uvcg_extension_drop().
		 */
		mutex_lock(&opts->lock);
		/*
		 * XUs can have an arbitrary string descriptor describing them. If they
		 * have one pick up the ID.
		 */
		list_for_each_entry(xu, &opts->extension_units, list)
			if (xu->string_descriptor_index)
				xu->desc.iExtension = cdev->usb_strings[xu->string_descriptor_index].id;
		/*
		 * We attach the hard-coded defaults incase the user does not provide
		 * any more appropriate strings through configfs.
		 */
		uvc_en_us_strings[UVC_STRING_CONTROL_IDX].s = opts->function_name;
		us = usb_gstrings_attach(cdev, uvc_function_strings,
					 ARRAY_SIZE(uvc_en_us_strings));
		if (IS_ERR(us)) {
			ret = PTR_ERR(us);
			goto error_unlock;
		}
		uvc_iad.iFunction = opts->iad_index ? cdev->usb_strings[opts->iad_index].id :
				    us[UVC_STRING_CONTROL_IDX].id;
		uvc_streaming_intf_alt0.iInterface = opts->vs0_index ?
						     cdev->usb_strings[opts->vs0_index].id :
						     us[UVC_STRING_STREAMING_IDX].id;
		uvc_streaming_intf_alt1.iInterface = opts->vs1_index ?
						     cdev->usb_strings[opts->vs1_index].id :
						     us[UVC_STRING_STREAMING_IDX].id;
		/* Allocate interface IDs. */
		if ((ret = usb_interface_id(c, f)) < 0)
			goto error_unlock;
		uvc_iad.bFirstInterface = ret;
		uvc_control_intf.bInterfaceNumber = ret;
		uvc->control_intf = ret;
		opts->control_interface = ret;
		if ((ret = usb_interface_id(c, f)) < 0)
			goto error_unlock;
		uvc_streaming_intf_alt0.bInterfaceNumber = ret;
		uvc_streaming_intf_alt1.bInterfaceNumber = ret;
		uvc->streaming_intf = ret;
		opts->streaming_interface = ret;
		/* Copy descriptors */
		f->fs_descriptors = uvc_copy_descriptors(uvc, USB_SPEED_FULL);
		if (IS_ERR(f->fs_descriptors)) {
			ret = PTR_ERR(f->fs_descriptors);
			f->fs_descriptors = NULL;
			goto error_unlock;
		}
		f->hs_descriptors = uvc_copy_descriptors(uvc, USB_SPEED_HIGH);
		if (IS_ERR(f->hs_descriptors)) {
			ret = PTR_ERR(f->hs_descriptors);
			f->hs_descriptors = NULL;
			goto error_unlock;
		}
		f->ss_descriptors = uvc_copy_descriptors(uvc, USB_SPEED_SUPER);
		if (IS_ERR(f->ss_descriptors)) {
			ret = PTR_ERR(f->ss_descriptors);
			f->ss_descriptors = NULL;
			goto error_unlock;
		}
		f->ssp_descriptors = uvc_copy_descriptors(uvc, USB_SPEED_SUPER_PLUS);
		if (IS_ERR(f->ssp_descriptors)) {
			ret = PTR_ERR(f->ssp_descriptors);
			f->ssp_descriptors = NULL;
			goto error_unlock;
		}
		mutex_unlock(&opts->lock);
		/* Preallocate control endpoint request. */
		uvc->control_req = usb_ep_alloc_request(cdev->gadget->ep0, GFP_KERNEL);
		uvc->control_buf = kmalloc(UVC_MAX_REQUEST_SIZE, GFP_KERNEL);
		if (uvc->control_req == NULL || uvc->control_buf == NULL) {
			ret = -ENOMEM;
			goto error;
		}
		uvc->control_req->buf = uvc->control_buf;
		uvc->control_req->complete = uvc_function_ep0_complete;
		uvc->control_req->context = uvc;
		if (v4l2_device_register(&cdev->gadget->dev, &uvc->v4l2_dev)) {
			uvcg_err(f, "failed to register V4L2 device\n");
			goto error;
		}
		/* Initialise video. */
		ret = uvcg_video_init(&uvc->video, uvc);
		if (ret < 0)
			goto v4l2_error;
		/* Register a V4L2 device. */
		ret = uvc_register_video(uvc);
		if (ret < 0) {
			uvcg_err(f, "failed to register video device\n");
			goto v4l2_error;
		}
		return 0;
	error_unlock:
		mutex_unlock(&opts->lock);
	v4l2_error:
		v4l2_device_unregister(&uvc->v4l2_dev);
	error:
		if (uvc->control_req)
			usb_ep_free_request(cdev->gadget->ep0, uvc->control_req);
		kfree(uvc->control_buf);
		usb_free_all_descriptors(f);
		return ret;
	}

The ``error:`` block will be the most important here:

.. code-block:: c
   error:
      if (uvc->control_req)
         usb_ep_free_request(cdev->gadget->ep0, uvc->control_req);
         //uvc->control_req is freed, but not assigned as NULL. Dangling Pointer!!
      kfree(uvc->control_buf);
      //uvc->control_buf freed, but not assigned as NULL. Dangling Pointer!!
      usb_free_all_descriptors(f);
      return ret;

As seen in the comments, both of these pointers are only freed, but are not nulled
contrary to kernel rules. we will discuss the fix later, moving on to the unbind
function ``uvc_function_unbind()``, where this is more blatant:

.. code-block:: c
   static void uvc_function_unbind(struct usb_configuration *c,
   				struct usb_function *f)
   {
   	DECLARE_COMPLETION_ONSTACK(vdev_release_done);
   	struct usb_composite_dev *cdev = c->cdev;
   	struct uvc_device *uvc = to_uvc(f);
   	struct uvc_video *video = &uvc->video;
   	long wait_ret = 1;
   	bool connected;
   	uvcg_info(f, "%s()\n", __func__);
   	scoped_guard(mutex, &uvc->lock) {
   		uvc->func_unbound = true;
   		uvc->vdev_release_done = &vdev_release_done;
   		connected = uvc->func_connected;
   	}
   	kthread_cancel_work_sync(&video->hw_submit);
   	if (video->async_wq)
   		destroy_workqueue(video->async_wq);
   	/*
   	 * If we know we're connected via v4l2, then there should be a cleanup
   	 * of the device from userspace either via UVC_EVENT_DISCONNECT or
   	 * though the video device removal uevent. Allow some time for the
   	 * application to close out before things get deleted.
   	 */
   	if (connected) {
   		uvcg_dbg(f, "waiting for clean disconnect\n");
   		wait_ret = wait_event_interruptible_timeout(uvc->func_connected_queue,
   				uvc->func_connected == false, msecs_to_jiffies(500));
   		uvcg_dbg(f, "done waiting with ret: %ld\n", wait_ret);
   	}
   	device_remove_file(&uvc->vdev.dev, &dev_attr_function_name);
   	video_unregister_device(&uvc->vdev);
   	v4l2_device_unregister(&uvc->v4l2_dev);
   	scoped_guard(mutex, &uvc->lock)
   		connected = uvc->func_connected;
   	if (connected) {
   		/*
   		 * Wait for the release to occur to ensure there are no longer any
   		 * pending operations that may cause panics when resources are cleaned
   		 * up.
   		 */
   		uvcg_warn(f, "%s no clean disconnect, wait for release\n", __func__);
   		wait_ret = wait_event_interruptible_timeout(uvc->func_connected_queue,
   				uvc->func_connected == false, msecs_to_jiffies(1000));
   		uvcg_dbg(f, "done waiting for release with ret: %ld\n", wait_ret);
   	}
   	/* Wait for the video device to be released */
   	wait_for_completion(&vdev_release_done);
   	uvc->vdev_release_done = NULL;
   	usb_ep_free_request(cdev->gadget->ep0, uvc->control_req);
      //Similar to bind failure path, usb->control_req is not nulled
   	kfree(uvc->control_buf);
      //usb->control_buf is freed but not nulled either
   	usb_free_all_descriptors(f);
   }

Both have the same mechanism. If the following scenario are attempted,
a UAF  and/or double free will occur:
1. We bind and unbind successfully, but fail on the second bind

The if condition is also useless in the error path of bind function as
prior to the patch, it is NOT nulled, it holds an address to freed data.

``if (uvc->control_req) //Addess points to freed data, Not null(0x0)``

The Solution
============

We can implement an easy fix that is seen everywhere else in the kernel.
Null the pointer once the address it points to has been freed. This
involved setting uvc->control_req and uvc->control_buf to NULL after
kfree.

The Patch
=========

As per solution, we implement patch as:

.. code-block:: diff

   diff --git a/drivers/usb/gadget/function/f_uvc.c b/drivers/usb/gadget/function/f_uvc.c
   index 73dc7e4..d1bf3ea 100644
   --- a/drivers/usb/gadget/function/f_uvc.c
   +++ b/drivers/usb/gadget/function/f_uvc.c
   @@ -889,9 +889,12 @@ uvc_function_bind(struct usb_configuration *c, struct usb_function *f)
      v4l2_error:
 	   v4l2_device_unregister(&uvc->v4l2_dev);
    error:
   -	if (uvc->control_req)
   +	if (uvc->control_req) {
 		   usb_ep_free_request(cdev->gadget->ep0, uvc->control_req);
   +		uvc->control_req = NULL;
   +	}
 	   kfree(uvc->control_buf);
   +	uvc->control_buf = NULL;
 
 	   usb_free_all_descriptors(f);
 	   return ret;
   @@ -1075,7 +1078,9 @@ static void uvc_function_unbind(struct usb_configuration *c,
 	   uvc->vdev_release_done = NULL;
 
 	   usb_ep_free_request(cdev->gadget->ep0, uvc->control_req);
   +	uvc->control_req = NULL;
 	   kfree(uvc->control_buf);
   +	uvc->control_buf = NULL;
 
 	   usb_free_all_descriptors(f);
   }

``Status: Not Merged(In usb-next)``

References
==========
https://patch.msgid.link/20260813174311.130823-1-jeffinphilip14@gmail.com
