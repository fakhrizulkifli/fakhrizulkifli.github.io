---
layout: post
title: "Libemu - Divide By Zero"
date: 2020-03-24
categories:
    - blog
keywords: "libemu, sctest, divide by zero, dos, denial of service, dionaea, scdbg"
---

I found this bug some time ago when i was reading the code for GetPC heuristics mechanism. The bug can be triggered just only with one byte which can be translated to a `CALL` opcode. In normal cases, `CALL` opcode must be supplied with a relative offset to a new procedure but in this case if the offset is not supplied, it triggers the bug.

The root cause is during the instructions callgraph creation. The callgraph will be created when it detects a branching opcode.

```c
File: src/emu_source.c
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
```

`emu_source_instruction_graph_create()` function stores the information about the instructions inside a hash table. At line 47, the hash table is created with a size of the instructions divide by 2. Then `datasize` is initialized to `eh->size`.

```c
File: src/emu_hashtable.c
094: struct emu_hashtable_item *emu_hashtable_search(struct emu_hashtable *eh, void *key)
095: {
096: 	uint32_t first_hash = eh->hash(key) % eh->size;     <-- triggered
097:
098: 	struct emu_hashtable_bucket *ehb = 	eh->buckets[first_hash];
099: 	if (ehb == NULL)
100: 		return NULL;
```

Going down the rabbit hole until the data is insert into the hash table at line 70. `emu_hashtable_search()` is called to check whether the data is already inserted. At line 96, the address of the key inside the hash table is divided with `eh->size`.