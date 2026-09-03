===========================================================
Null Pointer Dereference in ``usb_put_function_instance()``
===========================================================

Introduction
============

What is a Null Pointer Dereference?
-----------------------------------

A Null Pointer Dereference is a error that occurs when you attempt 
to dereference a pointer whose address is 0x0(NULL). An example would
that causes a null-ptr-deref would look like this:

.. code-block:: c

	int function() {
		*ptr = NULL;
		int var = ptr->member; //ptr is NULL here.
	}

The Problem
===========

It all starts in ``try_get_usb_function_instance()``. Here is the function:

.. code-block:: c

	static struct usb_function_instance *try_get_usb_function_instance(const char *name)
	{
		struct usb_function_driver *fd;
		struct usb_function_instance *fi;

		fi = ERR_PTR(-ENOENT);
		mutex_lock(&func_lock);
		list_for_each_entry(fd, &func_list, list) {

			if (strcmp(name, fd->name))
				continue;

			if (!try_module_get(fd->mod)) {
				fi = ERR_PTR(-EBUSY);
				break;
			}
	>>		fi = fd->alloc_inst(); //Allocation here
			if (IS_ERR(fi))
				module_put(fd->mod);
			else
				fi->fd = fd; //fi->fd allocated here
			break;
		}
		mutex_unlock(&func_lock);
		return fi;
	}

Notice where fi is allocated and fi->fd is allocated. This will be crucial.
We will then move on the the allocation function. 

.. code-block:: c

	static struct usb_function_instance *uvc_alloc_inst(void)
	{
		struct f_uvc_opts *opts;
		struct uvc_camera_terminal_descriptor *cd;
		struct uvc_processing_unit_descriptor *pd;
		struct uvc_output_terminal_descriptor *od;
		struct uvc_descriptor_header **ctl_cls;
		int ret;

		opts = kzalloc_obj(*opts);
		if (!opts)
			return ERR_PTR(-ENOMEM);
		opts->func_inst.free_func_inst = uvc_free_inst;
		mutex_init(&opts->lock);

		cd = &opts->uvc_camera_terminal;
		cd->bLength			= UVC_DT_CAMERA_TERMINAL_SIZE(3);
		cd->bDescriptorType		= USB_DT_CS_INTERFACE;
		cd->bDescriptorSubType		= UVC_VC_INPUT_TERMINAL;
		cd->bTerminalID			= 1;
		cd->wTerminalType		= cpu_to_le16(0x0201);
		cd->bAssocTerminal		= 0;
		cd->iTerminal			= 0;
		cd->wObjectiveFocalLengthMin	= cpu_to_le16(0);
		cd->wObjectiveFocalLengthMax	= cpu_to_le16(0);
		cd->wOcularFocalLength		= cpu_to_le16(0);
		cd->bControlSize		= 3;
		cd->bmControls[0]		= 2;
		cd->bmControls[1]		= 0;
		cd->bmControls[2]		= 0;

		pd = &opts->uvc_processing;
		pd->bLength			= UVC_DT_PROCESSING_UNIT_SIZE(2);
		pd->bDescriptorType		= USB_DT_CS_INTERFACE;
		pd->bDescriptorSubType		= UVC_VC_PROCESSING_UNIT;
		pd->bUnitID			= 2;
		pd->bSourceID			= 1;
		pd->wMaxMultiplier		= cpu_to_le16(16*1024);
		pd->bControlSize		= 2;
		pd->bmControls[0]		= 1;
		pd->bmControls[1]		= 0;
		pd->iProcessing			= 0;
		pd->bmVideoStandards		= 0;

		od = &opts->uvc_output_terminal;
		od->bLength			= UVC_DT_OUTPUT_TERMINAL_SIZE;
		od->bDescriptorType		= USB_DT_CS_INTERFACE;
		od->bDescriptorSubType		= UVC_VC_OUTPUT_TERMINAL;
		od->bTerminalID			= 3;
		od->wTerminalType		= cpu_to_le16(0x0101);
		od->bAssocTerminal		= 0;
		od->bSourceID			= 2;
		od->iTerminal			= 0;

		/*
		 * With the ability to add XUs to the UVC function graph, we need to be
		 * able to allocate unique unit IDs to them. The IDs are 1-based, with
		 * the CT, PU and OT above consuming the first 3.
		 */
		opts->last_unit_id		= 3;

		/* Prepare fs control class descriptors for configfs-based gadgets */
		ctl_cls = opts->uvc_fs_control_cls;
		ctl_cls[0] = NULL;	/* assigned elsewhere by configfs */
		ctl_cls[1] = (struct uvc_descriptor_header *)cd;
		ctl_cls[2] = (struct uvc_descriptor_header *)pd;
		ctl_cls[3] = (struct uvc_descriptor_header *)od;
		ctl_cls[4] = NULL;	/* NULL-terminate */
		opts->fs_control =
			(const struct uvc_descriptor_header * const *)ctl_cls;

		/* Prepare hs control class descriptors for configfs-based gadgets */
		ctl_cls = opts->uvc_ss_control_cls;
		ctl_cls[0] = NULL;	/* assigned elsewhere by configfs */
		ctl_cls[1] = (struct uvc_descriptor_header *)cd;
		ctl_cls[2] = (struct uvc_descriptor_header *)pd;
		ctl_cls[3] = (struct uvc_descriptor_header *)od;
		ctl_cls[4] = NULL;	/* NULL-terminate */
		opts->ss_control =
			(const struct uvc_descriptor_header * const *)ctl_cls;

		INIT_LIST_HEAD(&opts->extension_units);

		opts->streaming_interval = 1;
		opts->streaming_maxpacket = 1024;
		snprintf(opts->function_name, sizeof(opts->function_name), "UVC Camera");

	>>	ret = uvcg_attach_configfs(opts);
		if (ret < 0) {
			kfree(opts);
			return ERR_PTR(ret);
		}

		return &opts->func_inst;
	}

Notice we still do not have fi->fd allocated yet. fi is still in the midst of being 
allocated itself. We move on to ``uvcg_attach_configfs()``:

.. code-block:: c

	int uvcg_attach_configfs(struct f_uvc_opts *opts)
	{
		int ret;

		config_group_init_type_name(&opts->func_inst.group, uvc_func_type.name,
					    &uvc_func_type.type);

		ret = uvcg_config_create_children(&opts->func_inst.group,
						  &uvc_func_type);
	>>	if (ret < 0)
			config_group_put(&opts->func_inst.group);

		return ret;
	}

We will take the failure path, which goes to ``config_group_put->config_item_put
->kref_put->config_item_release->config_item_cleanup->usb_put_function_instance``.
In ``usb_put_function_instance``, we have:

.. code-block:: c

	void usb_put_function_instance(struct usb_function_instance *fi)
	{
		struct module *mod;

		if (!fi) //fi is checked for NULL correctly
			return;

	>>	mod = fi->fd->mod; //fi->fd is not checked for NULL, null-ptr-deref here
		fi->free_func_inst(fi);
		module_put(mod);
	}

We have seen from the start that fi->fd cannot be allocated unless fi is allocated
first as a child cannot be allocated before the parent struct. Here, we do not follow
that rule and attemopt to dereference a child in a parent struct that is not even
allocated fully yet. 

The Solution
============

This can be fixed by checking for fi->fd value too and returning if it is NULL, similar
to null check for fi.

The Patch
=========

Based on the solution above, we patch this issue by:

.. code-block:: diff

	diff --git a/drivers/usb/gadget/functions.c b/drivers/usb/gadget/functions.c
	index 203361a64212..70e31c40e267 100644
	--- a/drivers/usb/gadget/functions.c
	+++ b/drivers/usb/gadget/functions.c
	@@ -70,7 +70,7 @@ void usb_put_function_instance(struct usb_function_instance *fi)
	 {
	 	struct module *mod;
	 
	-	if (!fi)
	+	if (!fi || !fi->fd)
	 		return;
	 
	 	mod = fi->fd->mod;
 
``Status: Pending Merge``

References
==========

https://patch.msgid.link/20260816061712.15547-1-jeffinphilip14@gmail.com
