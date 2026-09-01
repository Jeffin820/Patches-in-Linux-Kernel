========================================================
Null Pointer Dereference in fsg_common_set_num_buffers()
========================================================

Introduction
============

What does fsg_common_set_num_buffers() do?
------------------------------------------

The function fsg_common_set_num_buffers allocates and configures the data
transfer buffer heads (and their underlying I/O buffers) used by the
File-Backed Storage gadget / Mass Storage Function (f_mass_storage) driver.

The Problem
===========

In ``fsg_common_set_num_buffers()``, we set buffhds to be allocated with n
objects using ``kzalloc_objs(*buffhds, n)`` where n is of type unsigned int
with a u8 type meaning its value can be from 0-255. However, there are 
certain issues with this statement. One, if n = 0, we risk a Null pointer 
dereference and since n is assigned using ``kstrtou8()`` in 
fsg_opts_num_buffers_store() and passed to our faulty function, it allocates
buffhds, this also passes the NULL check in ``if(!buffhds)`` as ``buffhds`` 
does not return NULL for n = 0, rather it outputs something called a
ZERO_SIZE_PTR, which has an address of (void *)16. This continues on to
``bh = buffhds`` and finally to ``bh->next`` where the NULL value is
dereferenced. The full function is below:

.. code-block:: c

   int fsg_common_set_num_buffers(struct fsg_common *common, unsigned int n)
   {
   	struct fsg_buffhd *bh, *buffhds;
   	int i;

   	buffhds = kzalloc_objs(*buffhds, n); //n is zero, and buffhds returns ZERO_SIZE_PTR
   	if (!buffhds) //Passes check as buffhds is TECHNICALLY NOT null
   		return -ENOMEM;

   	/* Data buffers cyclic list */
   	bh = buffhds;
   	i = n;
   	goto buffhds_first_it;
   	do {
   		bh->next = bh + 1;
   		++bh;
   buffhds_first_it:
   		bh->buf = kmalloc(FSG_BUFLEN, GFP_KERNEL);
   		if (unlikely(!bh->buf))
   			goto error_release;
   	} while (--i);
   	bh->next = buffhds;

   	_fsg_common_free_buffers(common->buffhds, common->fsg_num_buffers);
   	common->fsg_num_buffers = n;
   	common->buffhds = buffhds;

   	return 0;

   error_release:
   	/*
   	 * "buf"s pointed to by heads after n - i are NULL
   	 * so releasing them won't hurt
   	 */
   	_fsg_common_free_buffers(buffhds, n);

   	return -ENOMEM;
   }

Now, for n to be 0, there is one way, ``kstrtou8()`` converts a string to
a u8 integer. However, if the string was ``0x0``, the u8 integer also was
converted to 0. This n value is then passed to ``fsg_common_set_num_buffers()``.

.. code-block:: c

   static ssize_t fsg_opts_num_buffers_show(struct config_item *item, char *page) //page is 0x0
   {
   	struct fsg_opts *opts = to_fsg_opts(item);
   	int result;

   	mutex_lock(&opts->lock);
   	result = sprintf(page, "%d", opts->common->fsg_num_buffers);
   	mutex_unlock(&opts->lock);

   	return result;
   }

   static ssize_t fsg_opts_num_buffers_store(struct config_item *item,
   					  const char *page, size_t len)
   {
   	struct fsg_opts *opts = to_fsg_opts(item);
   	int ret;
   	u8 num;

   	mutex_lock(&opts->lock);
   	if (opts->refcnt) {
   		ret = -EBUSY;
   		goto end;
   	}
   	ret = kstrtou8(page, 0, &num); //Success, but n = 0
   	if (ret)
   		goto end;

   	ret = fsg_common_set_num_buffers(opts->common, num); //Target function called with n = 0
   	if (ret)
   		goto end;
   	ret = len;

   end:
   	mutex_unlock(&opts->lock);
   	return ret;
   }

Thus, we get the null pointer dereference.

The Solution
============

To find the solution, we must move back in history to see an older commit.
Commit fe5a6c48fd95 removed a function ``fsg_num_buffers_validate()`` and
stated the reason as "valid range for storage buffers is encoded in
``Kconfig`` already. Instead of checking again, let's drop ``fsg_num_buffers_validate()`` 
altogether." Looking at ``fsg_num_buffers_validate()``:

.. code-block:: c

   static inline int fsg_num_buffers_validate(unsigned int fsg_num_buffers)
   {
   #define FSG_MAX_NUM_BUFFERS    32

          if (fsg_num_buffers >= 2 && fsg_num_buffers <= FSG_MAX_NUM_BUFFERS)
                  return 0;
          pr_err("fsg_num_buffers %u is out of range (%d to %d)\n",
                 fsg_num_buffers, 2, FSG_MAX_NUM_BUFFERS);
          return -EINVAL;
   }

In ``Kconfig``, we see:

.. code-block:: kconfig

   config USB_GADGET_STORAGE_NUM_BUFFERS
	   int "Number of storage pipeline buffers"
	   range 2 256
	   default 2

Our range goes from 2 - 256 with default as 2, this is consistent with
``fsg_num_buffers_validate()`` which also take 2 as the lower limit. u8 also
caps at 255 which is below our range that is 256 here. However, that
commit forgot the edge cases of n = 0 and n = 1, leading to our null
pointer dereference, we can fix it by just adding a check for n != 0 or 1
consistent with ``Kconfig`` and ``fsg_num_buffers_validate()``

The Patch
=========

Based on our solution, we can patch this by:

.. code-block:: diff

   diff --git a/drivers/usb/gadget/function/f_mass_storage.c b/drivers/usb/gadget/function/f_mass_storage.c
   index a50743caf083..4e743be220cc 100644
   --- a/drivers/usb/gadget/function/f_mass_storage.c
   +++ b/drivers/usb/gadget/function/f_mass_storage.c
   @@ -2747,6 +2747,9 @@ int fsg_common_set_num_buffers(struct fsg_common *common, unsigned int n)
    	struct fsg_buffhd *bh, *buffhds;
    	int i;
    
   +	if (n < 2)
   +		return -EINVAL;
   +
    	buffhds = kzalloc_objs(*buffhds, n);
    	if (!buffhds)
    		return -ENOMEM;

``Status: Pending merge(v7.3-rc2)``

References
==========
https://patch.msgid.link/20260818035904.10324-1-jeffinphilip14@gmail.com
