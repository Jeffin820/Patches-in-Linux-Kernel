======================================
Null-Pointer-Dereference-in-UVC-Gadget
======================================

Introduction
============

What is the UVC Gadget driver?
------------------------------

The UVC Gadget driver is a driver for hardware on the device side of a USB connection.
It is intended to run on a Linux system that has USB device-side hardware such as boards
with an OTG port.(*taken from kernel docs*)

The Problem
===========

In video initialization function ```uvcg_video_init()``` error path, we use
```uvcg_err(&video->uvc->func, "failed to create UVCG kworker\n");```, however, if you
look at the whole function:

..code-block:: c

int uvcg_video_init(struct uvc_video *video, struct uvc_device *uvc)
{
	video->is_enabled = false;
	INIT_LIST_HEAD(&video->ureqs);
	INIT_LIST_HEAD(&video->req_free);
	INIT_LIST_HEAD(&video->req_ready);
	spin_lock_init(&video->req_lock);
	INIT_WORK(&video->pump, uvcg_video_pump);
	/* Allocate a work queue for asynchronous video pump handler. */
	video->async_wq = alloc_workqueue("uvcgadget", WQ_UNBOUND | WQ_HIGHPRI, 0);
	if (!video->async_wq)
		return -EINVAL;
	/* Allocate a kthread for asynchronous hw submit handler. */
	video->kworker = kthread_run_worker(0, "UVCG");
	if (IS_ERR(video->kworker)) {
		uvcg_err(&video->uvc->func, "failed to create UVCG kworker\n"); **//video->uvc never allocated, allocated much later**
		return PTR_ERR(video->kworker);
	}
	kthread_init_work(&video->hw_submit, uvcg_video_hw_submit);
	sched_set_fifo(video->kworker->task);
	video->uvc = uvc; **//video->uvc allocated here**
	video->fcc = V4L2_PIX_FMT_YUYV;
	video->bpp = 16;
	video->width = 320;
	video->height = 240;
	video->imagesize = 320 * 240 * 2;
	video->interval = 666666;
	/* Initialize the video buffers queue. */
	return uvcg_queue_init(&video->queue, uvc->v4l2_dev.dev->parent,
			V4L2_BUF_TYPE_VIDEO_OUTPUT, &video->mutex);
}

```vide->uvc``` is referenced before it is allocated, which means uvc could by ```NULL```
and we have a textbook example of null pointer dereference.

The Solution
============

To fix this, we first need to look at the video struct, it is of type ```struct uvc_video```.
Looking at its blueprint(*taken from kernel docs*), we have:
..code-block:: c

struct uvc_video {
	struct uvc_device *uvc; //TARGET STRUCT
	struct usb_ep *ep;

	struct work_struct pump;
	struct workqueue_struct *async_wq;

	struct kthread_worker*kworker;
	struct kthread_work  hw_submit;

	atomic_t queued;

	/* Frame parameters */
	u8 bpp;
	u32 fcc;
	unsigned int width;
	unsigned int height;
	unsigned int imagesize;
	unsigned int interval;	/* in 100ns units */
	struct mutex mutex;	/* protects frame parameters */

	unsigned int uvc_num_requests;

	unsigned int reqs_per_frame;

	/* Requests */
	bool is_enabled; /* tracks whether video stream is enabled */
	unsigned int req_size;
	unsigned int max_req_size;
	struct list_head ureqs; /* all uvc_requests allocated by uvc_video */

	/* USB requests that the video pump thread can encode into */
	struct list_head req_free;

	/*
	 * USB requests video pump thread has already encoded into. These are
	 * ready to be queued to the endpoint.
	 */
	struct list_head req_ready;
	spinlock_t req_lock;

	unsigned int req_int_count;

	void (*encode) (struct usb_request *req, struct uvc_video *video,
			struct uvc_buffer *buf);

	/* Context data used by the completion handler */
	__u32 payload_size;
	__u32 max_payload_size;

	struct uvc_video_queue queue;
	unsigned int fid;
};

We see ```uvc``` is of type ```struct uvc_device```. We also get uvc as an argument that is
passed in ```uvcg_video_init()```. This is especially useful as we can now just use
```uvc``` directly without referencing ```video```. Looking at ```struct uvc_device```, we
see:
.. code-block:: c

struct uvc_device {
	struct video_device vdev;
	struct v4l2_device v4l2_dev;
	enum uvc_state state;
	struct usb_function func; //TARGET STRUCT
	struct uvc_video video;
	struct completion *vdev_release_done;
	struct mutex lock;	/* protects func_unbound and func_connected */
	bool func_unbound;
	bool func_connected;
	wait_queue_head_t func_connected_queue;

	struct uvcg_streaming_header *header;

	/* Descriptors */
	struct {
		const struct uvc_descriptor_header * const *fs_control;
		const struct uvc_descriptor_header * const *ss_control;
		const struct uvc_descriptor_header * const *fs_streaming;
		const struct uvc_descriptor_header * const *hs_streaming;
		const struct uvc_descriptor_header * const *ss_streaming;
		struct list_head *extension_units;
	} desc;

	unsigned int control_intf;
	struct usb_ep *interrupt_ep;
	struct usb_request *control_req;
	void *control_buf;
	bool enable_interrupt_ep;

	unsigned int streaming_intf;

	/* Events */
	unsigned int event_length;
	unsigned int event_setup_out : 1;
};

```func``` is directly present in uvc. Hence we can use the ```uvc``` passed as argument to our
```uvcg_video_init()``` function and resolve the Null Pointer Dereference.

The Patch
=========

As per solution, we implement the patch as:

..code-block:: diff
diff --git a/drivers/usb/gadget/function/uvc_video.c b/drivers/usb/gadget/function/uvc_video.c
index 2f9700b..9ba0911 100644
--- a/drivers/usb/gadget/function/uvc_video.c
+++ b/drivers/usb/gadget/function/uvc_video.c
@@ -821,7 +821,7 @@ int uvcg_video_init(struct uvc_video *video, struct uvc_device *uvc)
 	/* Allocate a kthread for asynchronous hw submit handler. */
 	video->kworker = kthread_run_worker(0, "UVCG");
 	if (IS_ERR(video->kworker)) {
-		uvcg_err(&video->uvc->func, "failed to create UVCG kworker\n");
+		uvcg_err(&uvc->func, "failed to create UVCG kworker\n");
 		return PTR_ERR(video->kworker);
 	}

```Status: *Not merged(Currently in usb-next)*```

References
========== 
<https://patch.msgid.link/20260804034338.7976-1-jeffinphilip14@gmail.com>
