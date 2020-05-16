---
layout: post
title: "Libemu - emu_shellcode_test Divide By Zero"
excerpt: "Dionaea remote denial of service."
modified: 2020-03-24
categories: blog
tags: ["libemu"]
comments: true
---

I found this bug some time ago when i was reading the code for GetPC heuristics mechanism. The bug can be triggered with just only one byte which can be translated to a **CALL** opcode. In normal cases, **CALL** opcode must be supplied with a relative offset to a new procedure but in this case if the offset is not supplied, it triggers the bug.

# Proof-of-Concept
{% highlight bash %}
$ sctest -gS < <(echo "\xe8")
{% endhighlight %}

# Bug Analysis
The root cause is when it detects a branching opcode it will attempt to create the instructions call graph by enumerating the relative offset for the next branch. However, there is no check whether the branching offset is supplied or not and thus leaving the `datasize` variable with value 0.

{% highlight c %}
041: uint32_t emu_source_instruction_graph_create(struct emu *e, struct emu_track_and_source *es, uint32_t datastart, uint32_t datasize)
042: {
043: //	printf("tracking from %x to %x\n", datastart, datastart+datasize);
044: 	struct emu_cpu *c = emu_cpu_get(e);
045:
046: 	es->static_instr_graph = emu_graph_new();
047: 	es->static_instr_table = emu_hashtable_new(datasize/2, emu_hashtable_ptr_hash,  emu_hashtable_ptr_cmp);     <-- datasize=0
048: 	es->static_instr_graph->vertex_destructor = emu_source_and_track_instr_info_free_void;
049:
050: 	uint32_t i;
051: 	for (i=datastart;i<datastart+datasize;i++)
052: 	{
053: 		emu_cpu_eip_set(c, i);
054:
055: 		if ( emu_cpu_parse(c) != 0)
056: 		{
057: //			printf("parse error %s\n", emu_strerror(e));
058: 			continue;
059: 		}
060:
061: 		if ( emu_cpu_step(c) != 0)
062: 		{
063: //			printf("step error %s\n", emu_strerror(e));
064: //			continue;
065: 		}
066:
067:         struct emu_source_and_track_instr_info *etii = emu_source_and_track_instr_info_new(c,i);
068: 		struct emu_vertex *ev = emu_vertex_new();
069: 		ev->data = etii;
070: 		emu_hashtable_insert(es->static_instr_table, (void *)(uintptr_t)i, ev);     <-- insert data
071: 		emu_graph_vertex_add(es->static_instr_graph, ev);
072: 	}
073:
{% endhighlight %}

**emu_source_instruction_graph_create** function is to stores the information about the instructions inside a hash table. At line 47, the hash table is created with the size of the instructions divided by 2. Then `datasize` is initialized to `eh->size`.

{% highlight c %}
094: struct emu_hashtable_item *emu_hashtable_search(struct emu_hashtable *eh, void *key)
095: {
096: 	uint32_t first_hash = eh->hash(key) % eh->size;     <-- triggered
097:
098: 	struct emu_hashtable_bucket *ehb = 	eh->buckets[first_hash];
099: 	if (ehb == NULL)
100: 		return NULL;
{% endhighlight %}

Going down the rabbit hole right before the data is insert into the hash table through **emu_hashtable_insert** at line 70. **emu_hashtable_search** is called to check whether the data is already inserted or not. At line 96, the address of the key inside the hash table will be divided with `eh->size` and triggers the bug.
