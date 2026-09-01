=====================================
use-after-free-in-dvb_device_remove()
=====================================

Introduction
============

What is a refcount?
-------------------

Refcount, also short for Reference Count is a metric
kept by the Linux kernel to know when to free memory
or a data structure that holds memory. Every operation
on a data structure either increments or decrements its
refcount. Usually a data structure is started with 
refcount = 1. This is done by a driver's probe() function.
Similarly, open(), write() and read() all bump the refcount
by 1. del() and put() drop the refcount by 1. When refcount
reaches 0, it is automatically freed. Operating on any 
memory that is freed leads to a UAF(Use-After-Free).

The Problem
===========

In dvb_device_remove(), ``dvb_device_put()`` is called. However,
its location violates kernel rules where ``put()`` must always
be the last after operating on all the struct's members. However,
in our function(``dvb_device_remove()``), we have:

.. code-block::c

   void dvb_remove_device(struct dvb_device *dvbdev)
   {
   	if (!dvbdev)
   		return;

   	down_write(&minor_rwsem);
   	dvb_minors[dvbdev->minor] = NULL;
   	dvb_device_put(dvbdev); //Freed here if refcount is 1(refcount dropped to 0)
   	up_write(&minor_rwsem);

   	dvb_media_device_free(dvbdev); //Attempt on dereferencing freed memory

   	device_destroy(dvb_class, MKDEV(DVB_MAJOR, dvbdev->minor));

   	list_del(&dvbdev->list_head);
   }
   EXPORT_SYMBOL(dvb_remove_device);

Here, if refcount is 1(which it likely is as we are in ``dvb_device_remove()``),
dvbdevice struct will be freed and ``dvb_media_device_free()`` will try to 
dereference the struct after it has been freed:

.. code-block::c

   static void dvb_media_device_free(struct dvb_device *dvbdev)
   {
   #if defined(CONFIG_MEDIA_CONTROLLER_DVB)
   	if (dvbdev->entity) { //UAF here. dvbdev struct is already freed
   		media_device_unregister_entity(dvbdev->entity);
   		kfree(dvbdev->entity);
   		kfree(dvbdev->pads);
   		dvbdev->entity = NULL;
   		dvbdev->pads = NULL;
   	}

   	if (dvbdev->tsout_entity) {
   		int i;

   		for (i = 0; i < dvbdev->tsout_num_entities; i++) {
   			media_device_unregister_entity(&dvbdev->tsout_entity[i]);
   			kfree(dvbdev->tsout_entity[i].name);
   		}
   		kfree(dvbdev->tsout_entity);
   		kfree(dvbdev->tsout_pads);
   		dvbdev->tsout_entity = NULL;
   		dvbdev->tsout_pads = NULL;

   		dvbdev->tsout_num_entities = 0;
   	}

   	if (dvbdev->intf_devnode) {
   		media_devnode_remove(dvbdev->intf_devnode);
   		dvbdev->intf_devnode = NULL;
   	}

   	if (dvbdev->adapter->conn) {
   		media_device_unregister_entity(dvbdev->adapter->conn);
   		kfree(dvbdev->adapter->conn);
   		dvbdev->adapter->conn = NULL;
   		kfree(dvbdev->adapter->conn_pads);
   		dvbdev->adapter->conn_pads = NULL;
   	}
   #endif
   }

The Solution
============

We have a very simple fix for this. Just move the ``put()`` function to
the end of ``dvb_device_remove()`` so members of ``dvbdev`` struct can be
safely freed and NULLed while dvbdev is alive and finally free ``dvbdev``.

The Patch
=========

As per solution, we implement the patch as:

.. code-block::diff

   diff --git a/drivers/media/dvb-core/dvbdev.c b/drivers/media/dvb-core/dvbdev.c
   index d753d329502a..9aa4b8fd7379 100644
   --- a/drivers/media/dvb-core/dvbdev.c
   +++ b/drivers/media/dvb-core/dvbdev.c
   @@ -598,7 +598,6 @@ void dvb_remove_device(struct dvb_device *dvbdev)
    
    	down_write(&minor_rwsem);
    	dvb_minors[dvbdev->minor] = NULL;
   -	dvb_device_put(dvbdev);
    	up_write(&minor_rwsem);
    
    	dvb_media_device_free(dvbdev);
   @@ -606,6 +605,8 @@ void dvb_remove_device(struct dvb_device *dvbdev)
    	device_destroy(dvb_class, MKDEV(DVB_MAJOR, dvbdev->minor));
    
    	list_del(&dvbdev->list_head);
   +
   +	dvb_device_put(dvbdev);
    }
    EXPORT_SYMBOL(dvb_remove_device);

``Status: Reviewed``

References
==========

https://lore.kernel.org/all/20260822112256.12346-1-jeffinphilip14@gmail.com/T/
