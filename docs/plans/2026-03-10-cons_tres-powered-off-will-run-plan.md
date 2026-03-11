# Cons_tres Will-Run Powered-Off Deferral Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** In `SELECT_MODE_WILL_RUN`, only consider powered-off nodes if no powered-on nodes can satisfy the job (including with overcommit), where powered-on includes `POWERING_UP` but excludes `POWERING_DOWN`.

**Architecture:** Add a two-pass will-run evaluation in `select/cons_tres`: first pass filters out `POWERED_DOWN` and `POWERING_DOWN` nodes and runs the existing will-run logic unchanged; if that fails, run the existing will-run logic with the original bitmap. This preserves current eligibility rules and only changes node selection order in will-run mode.

**Tech Stack:** C, Slurm select/cons_tres plugin, libcheck (slurm_unit)

---

### Task 1: Add failing will-run tests for powered-off deferral

**Files:**
- Modify: `testsuite/slurm_unit/backfill/backfill_test.c`
- Test: `testsuite/slurm_unit/backfill/backfill_test.c`

**Step 1: Add helper for restoring node state (local to tests)**

Add a small helper near the tests:

```c
static void _restore_node_state(int node_i, uint32_t state)
{
	node_record_table_ptr[node_i]->node_state = state;
}
```

**Step 2: Add test where powered-on can satisfy (powered-off excluded)**

```c
START_TEST(test_will_run_powered_off_deferred)
{
	job_record_t *job_ptr;
	bitstr_t *avail_map = bit_alloc(node_record_count);
	uint32_t min_nodes = 1, max_nodes = 1, req_nodes = 1;
	uint16_t mode = SELECT_MODE_WILL_RUN;
	part_record_t *part_ptr = find_part_record("test");
	uint32_t node0_state = node_record_table_ptr[0]->node_state;
	uint32_t node1_state = node_record_table_ptr[1]->node_state;

	part_ptr->max_share = 2; /* allow oversubscribe */

	node_record_table_ptr[0]->node_state |= NODE_STATE_POWERED_DOWN;
	node_record_table_ptr[1]->node_state &= ~NODE_STATE_POWERED_DOWN;
	node_record_table_ptr[1]->node_state &= ~NODE_STATE_POWERING_DOWN;
	node_record_table_ptr[1]->node_state |= NODE_STATE_POWERING_UP;

	bit_set(avail_map, 0);
	bit_set(avail_map, 1);

	job_ptr = __add_job(0, 10, 1, 1, 10, NULL);
	job_ptr->details->overcommit = 1;
	job_ptr->details->cpus_per_task = 1;

	ck_assert_int_eq(
		select_g_job_test(job_ptr, avail_map, min_nodes, max_nodes,
				  req_nodes, mode, NULL, NULL, NULL, NULL),
		SLURM_SUCCESS);

	ck_assert_msg(bit_test(avail_map, 1),
		"Powered-on node should be selected when it can satisfy");
	ck_assert_msg(!bit_test(avail_map, 0),
		"Powered-down node should be excluded when powered-on works");

	_restore_node_state(0, node0_state);
	_restore_node_state(1, node1_state);
	FREE_NULL_BITMAP(avail_map);
}
END_TEST
```

**Step 3: Add test where only powered-off can satisfy (fallback)**

```c
START_TEST(test_will_run_powered_off_fallback)
{
	job_record_t *job_ptr;
	bitstr_t *avail_map = bit_alloc(node_record_count);
	uint32_t min_nodes = 1, max_nodes = 1, req_nodes = 1;
	uint16_t mode = SELECT_MODE_WILL_RUN;
	uint32_t node0_state = node_record_table_ptr[0]->node_state;
	uint32_t node1_state = node_record_table_ptr[1]->node_state;

	node_record_table_ptr[0]->node_state |= NODE_STATE_POWERED_DOWN;
	node_record_table_ptr[1]->node_state &= ~NODE_STATE_POWERED_DOWN;
	node_record_table_ptr[1]->node_state &= ~NODE_STATE_POWERING_DOWN;
	node_record_table_ptr[1]->node_state &= ~NODE_STATE_POWERING_UP;
	node_record_table_ptr[1]->node_state |= NODE_STATE_DRAIN;

	bit_set(avail_map, 0);
	bit_set(avail_map, 1);

	job_ptr = __add_job(0, 10, 1, 1, 10, NULL);
	job_ptr->details->overcommit = 0;
	job_ptr->details->cpus_per_task = 1;

	ck_assert_int_eq(
		select_g_job_test(job_ptr, avail_map, min_nodes, max_nodes,
				  req_nodes, mode, NULL, NULL, NULL, NULL),
		SLURM_SUCCESS);

	ck_assert_msg(bit_test(avail_map, 0),
		"Powered-down node should be selected when powered-on fails");

	_restore_node_state(0, node0_state);
	_restore_node_state(1, node1_state);
	FREE_NULL_BITMAP(avail_map);
}
END_TEST
```

**Step 4: Add tests to the suite**

```c
		tcase_add_test(tc, test_will_run_powered_off_deferred);
		tcase_add_test(tc, test_will_run_powered_off_fallback);
```

**Step 5: Run test to verify it fails**

Run: `make -C testsuite/slurm_unit/backfill check`
Expected: FAIL in `test_will_run_powered_off_deferred` (powered-down node still eligible when powered-on can satisfy).

**Step 6: Commit**

```bash
git add testsuite/slurm_unit/backfill/backfill_test.c
git commit -m "tests: cover will-run powered-off deferral"
```

---

### Task 2: Implement two-pass will-run selection

**Files:**
- Modify: `src/plugins/select/cons_tres/job_test.c`

**Step 1: Add helper to filter powered-on nodes**

```c
static bitstr_t *_filter_powered_on_nodes(bitstr_t *node_bitmap)
{
	bitstr_t *filtered = bit_copy(node_bitmap);
	node_record_t *node_ptr;

	for (int i = 0; (node_ptr = next_node_bitmap(filtered, &i)); i++) {
		if (IS_NODE_POWERED_DOWN(node_ptr) ||
		    IS_NODE_POWERING_DOWN(node_ptr))
			bit_clear(filtered, i);
	}

	return filtered;
}
```

**Step 2: Extract existing will-run logic into a helper**

Move the current body of `_will_run_test()` into a new helper that accepts the bitmap to use:

```c
static int _will_run_test_with_bitmap(job_record_t *job_ptr,
				     bitstr_t *node_bitmap,
				     uint32_t min_nodes, uint32_t max_nodes,
				     uint32_t req_nodes, uint16_t job_node_req,
				     list_t *preemptee_candidates,
				     list_t **preemptee_job_list,
				     resv_exc_t *resv_exc_ptr,
				     will_run_data_t *will_run_ptr)
{
	/* Existing _will_run_test body goes here unchanged */
}
```

**Step 3: Implement two-pass wrapper in `_will_run_test()`**

```c
static int _will_run_test(job_record_t *job_ptr, bitstr_t *node_bitmap,
			  uint32_t min_nodes, uint32_t max_nodes,
			  uint32_t req_nodes, uint16_t job_node_req,
			  list_t *preemptee_candidates,
			  list_t **preemptee_job_list,
			  resv_exc_t *resv_exc_ptr,
			  will_run_data_t *will_run_ptr)
{
	int rc;
	bitstr_t *filtered = _filter_powered_on_nodes(node_bitmap);

	if (filtered && bit_set_count(filtered) > 0) {
		rc = _will_run_test_with_bitmap(job_ptr, filtered, min_nodes, max_nodes,
					  req_nodes, job_node_req,
					  preemptee_candidates, preemptee_job_list,
					  resv_exc_ptr, will_run_ptr);
		if (rc == SLURM_SUCCESS) {
			bit_copybits(node_bitmap, filtered);
			FREE_NULL_BITMAP(filtered);
			return rc;
		}
	}

	FREE_NULL_BITMAP(filtered);
	return _will_run_test_with_bitmap(job_ptr, node_bitmap, min_nodes,
					 max_nodes, req_nodes, job_node_req,
					 preemptee_candidates, preemptee_job_list,
					 resv_exc_ptr, will_run_ptr);
}
```

**Step 4: Run test to verify it passes**

Run: `make -C testsuite/slurm_unit/backfill check`
Expected: PASS for the two new tests.

**Step 5: Commit**

```bash
git add src/plugins/select/cons_tres/job_test.c
git commit -m "select/cons_tres: defer powered-off nodes in will-run"
```

---

### Task 3: Run unit suite for regressions

**Files:**
- Test: `testsuite/slurm_unit/backfill/backfill_test.c`

**Step 1: Run full unit tests**

Run: `make -C testsuite/slurm_unit check`
Expected: PASS

**Step 2: Commit (if any test adjustments were needed)**

```bash
git add testsuite/slurm_unit/backfill/backfill_test.c
git commit -m "tests: stabilize will-run powered-off deferral"
```

---

Plan complete and saved to `docs/plans/2026-03-10-cons_tres-powered-off-will-run-plan.md`. Two execution options:

1. Subagent-Driven (this session) - I dispatch fresh subagent per task, review between tasks, fast iteration
2. Parallel Session (separate) - Open new session with executing-plans, batch execution with checkpoints

Which approach?
