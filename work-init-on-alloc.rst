============================
Fix Warning in f_midi_free()
============================

Introduction
============

What does f_midi_free() do?
---------------------------

f_midi_free is a cleanup function that looks at a midi function 
object's refcount and deallocates and cleans up the object.

The Problem
===========

In ``f_midi_free()``, we attempt to lower our refcount and deallocate
our midi function object. However, the first thing it does after 
checking for our refcount is to ``cancel_work_sync()`` which cancels
any pending work. This is seen below:

.. code-block:: c

   static void f_midi_free(struct usb_function *f)
   {
   	struct f_midi *midi;
   	struct f_midi_opts *opts;
   	bool free = false;

   	midi = func_to_midi(f);
   	opts = container_of(f->fi, struct f_midi_opts, func_inst);
   	mutex_lock(&opts->lock);
   	if (!--midi->free_ref) { //decrement refcount
   		cancel_work_sync(&midi->work); //cancel work
   		kfree(midi->id);
   		kfifo_free(&midi->in_req_fifo);
   		kfree(midi);
   		free = true;
   	}
   	mutex_unlock(&opts->lock);

   	if (free)
   		f_midi_free_inst(&opts->func_inst);
   }   

However, ``&midi->work`` is not initialized at allocation, but rather during bind:

.. code-block:: c

   static int f_midi_bind(struct usb_configuration *c, struct usb_function *f)
   {
   	struct usb_descriptor_header **midi_function;
   	struct usb_midi_in_jack_descriptor jack_in_ext_desc[MAX_PORTS];
   	struct usb_midi_in_jack_descriptor jack_in_emb_desc[MAX_PORTS];
   	struct usb_midi_out_jack_descriptor_1 jack_out_ext_desc[MAX_PORTS];
   	struct usb_midi_out_jack_descriptor_1 jack_out_emb_desc[MAX_PORTS];
   	struct usb_composite_dev *cdev = c->cdev;
   	struct f_midi *midi = func_to_midi(f);
   	struct usb_string *us;
   	struct f_midi_opts *opts;
   	int status, n, jack = 1, i = 0, endpoint_descriptor_index = 0;

   	midi->gadget = cdev->gadget;
   	INIT_WORK(&midi->work, f_midi_in_work); //Initialized here
   	status = f_midi_register_card(midi);
   	if (status < 0)
   		goto fail_register;

   	opts = container_of(f->fi, struct f_midi_opts, func_inst);
   	if (opts->interface_string)
   		midi_string_defs[STRING_FUNC_IDX].s = opts->interface_string;

   	/* maybe allocate device-global string ID */
   	us = usb_gstrings_attach(c->cdev, midi_strings,
   				 ARRAY_SIZE(midi_string_defs));
   	if (IS_ERR(us)) {
   		status = PTR_ERR(us);
   		goto fail;
   	}

      //Irrelevant

Similarly, midi->free_ref is also initialized to 1 during allocation:

.. code-block:: c

   static struct usb_function *f_midi_alloc(struct usb_function_instance *fi)
   {
   	struct f_midi *midi = NULL;
   	struct f_midi_opts *opts;
   	int status, i;

   	opts = container_of(fi, struct f_midi_opts, func_inst);

   	mutex_lock(&opts->lock);
   	/* sanity check */
   	if (opts->in_ports > MAX_PORTS || opts->out_ports > MAX_PORTS) {
   		status = -EINVAL;
   		goto setup_fail;
   	}

   	/* allocate and initialize one new instance */
   	midi = kzalloc_flex(*midi, in_ports_array, opts->in_ports);
   	if (!midi) {
   		status = -ENOMEM;
   		goto setup_fail;
   	}
   	midi->in_ports = opts->in_ports;

   	for (i = 0; i < opts->in_ports; i++)
   		midi->in_ports_array[i].cable = i;

   	/* set up ALSA midi devices */
   	midi->id = kstrdup(opts->id, GFP_KERNEL);
   	if (opts->id && !midi->id) {
   		status = -ENOMEM;
   		goto midi_free;
   	}
   	midi->out_ports = opts->out_ports;
   	midi->index = opts->index;
   	midi->buflen = opts->buflen;
   	midi->qlen = opts->qlen;
   	midi->in_last_port = 0;
   	midi->free_ref = 1; //Initialized to 1 here

      //Irrelevant

The only other place where refcount can be incremented
is in ``f_midi_register_card()`` which is in the bind
function:

.. code-block:: c

   static int f_midi_bind(struct usb_configuration *c, struct usb_function *f)
   {
   	struct usb_descriptor_header **midi_function;
   	struct usb_midi_in_jack_descriptor jack_in_ext_desc[MAX_PORTS];
   	struct usb_midi_in_jack_descriptor jack_in_emb_desc[MAX_PORTS];
   	struct usb_midi_out_jack_descriptor_1 jack_out_ext_desc[MAX_PORTS];
   	struct usb_midi_out_jack_descriptor_1 jack_out_emb_desc[MAX_PORTS];
   	struct usb_composite_dev *cdev = c->cdev;
   	struct f_midi *midi = func_to_midi(f);
   	struct usb_string *us;
   	struct f_midi_opts *opts;
   	int status, n, jack = 1, i = 0, endpoint_descriptor_index = 0;

   	midi->gadget = cdev->gadget;
   	INIT_WORK(&midi->work, f_midi_in_work);
   	status = f_midi_register_card(midi); //Used here

      //Irrelevant

In ``f_midi_register_card()``:

.. code-block:: c

   static int f_midi_register_card(struct f_midi *midi)
   {
   	struct snd_card *card;
   	struct snd_rawmidi *rmidi;
   	int err;
   	static struct snd_device_ops ops = {
   		.dev_free = f_midi_snd_free,
   	};

   	err = snd_card_new(&midi->gadget->dev, midi->index, midi->id,
   			   THIS_MODULE, 0, &card);
   	if (err < 0) {
   		ERROR(midi, "snd_card_new() failed\n");
   		goto fail;
   	}
   	midi->card = card;

   	err = snd_device_new(card, SNDRV_DEV_LOWLEVEL, midi, &ops);
   	if (err < 0) {
   		ERROR(midi, "snd_device_new() failed: error %d\n", err);
   		goto fail;
   	}

   	strscpy(card->driver, f_midi_longname);
   	strscpy(card->longname, f_midi_longname);
   	strscpy(card->shortname, f_midi_shortname);

   	/* Set up rawmidi */
   	snd_component_add(card, "MIDI");
   	err = snd_rawmidi_new(card, card->longname, 0,
   			      midi->out_ports, midi->in_ports, &rmidi);
   	if (err < 0) {
   		ERROR(midi, "snd_rawmidi_new() failed: error %d\n", err);
   		goto fail;
   	}
   	midi->rmidi = rmidi;
   	midi->in_last_port = 0;
   	strscpy(rmidi->name, card->shortname);
   	rmidi->info_flags = SNDRV_RAWMIDI_INFO_OUTPUT |
   			    SNDRV_RAWMIDI_INFO_INPUT |
   			    SNDRV_RAWMIDI_INFO_DUPLEX;
   	rmidi->private_data = midi;
   	rmidi->private_free = f_midi_rmidi_free;
   	midi->free_ref++; //Incremented here

      //Irrelevant

So, if bind is not run, refcount will be 1, if ``f_midi_free` is 
run immediately, we decrement refcount 1 is now down to 0 and we
``attempt cancel_work_sync()`` except, as seen before, work is
never initialized in the first place as ``INIT_WORK()`` is in
bind function and bind was never run. This is caught as a WARNING
in __flush_work later.

.. code-block:: c

   static bool __flush_work(struct work_struct *work, bool from_cancel)
   {
   	struct wq_barrier barr;

   	if (WARN_ON(!wq_online))
   		return false;

   	if (WARN_ON(!work->func)) //Caught here
   		return false;


The Solution
============

To fix this problem, we can initialize work in allocation function rather than
bind function as it doesn't make a difference anyway. ``cancel_work_sync()`` can
then run smoothly as work will be initialized long before refcount 1 becomes 0.

The Patch
=========

Base on the solution, our patch would be:

.. code-block:: diff

   diff --git a/drivers/usb/gadget/function/f_midi.c b/drivers/usb/gadget/function/f_midi.c
   index fba8cf787d6c..63fb6ee70a3d 100644
   --- a/drivers/usb/gadget/function/f_midi.c
   +++ b/drivers/usb/gadget/function/f_midi.c
   @@ -879,7 +879,6 @@ static int f_midi_bind(struct usb_configuration *c, struct usb_function *f)
    	int status, n, jack = 1, i = 0, endpoint_descriptor_index = 0;
    
    	midi->gadget = cdev->gadget;
   -	INIT_WORK(&midi->work, f_midi_in_work);
    	status = f_midi_register_card(midi);
    	if (status < 0)
    		goto fail_register;
   @@ -1377,6 +1376,7 @@ static struct usb_function *f_midi_alloc(struct usb_function_instance *fi)
    		status = -ENOMEM;
    		goto midi_free;
    	}
   +	INIT_WORK(&midi->work, f_midi_in_work);
    	midi->out_ports = opts->out_ports;
    	midi->index = opts->index;
    	midi->buflen = opts->buflen;

``Status: Reviewed``

References
==========

https://lore.kernel.org/all/20260815054006.102325-1-jeffinphilip14@gmail.com/
