# Cons_tres Mode Dispatch Refactor Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Simplify `job_test()` mode handling by centralizing powered-off deferral dispatch without changing scheduling behavior.

**Architecture:** Introduce a helper that handles mode dispatch and powered-off deferral. `SELECT_MODE_WILL_RUN` delegates directly to `_will_run_test()` (which already performs its own two-pass logic), while `SELECT_MODE_TEST_ONLY` and `SELECT_MODE_RUN_NOW` use a shared two-pass helper built around `_filter_powered_on_nodes()` and a small mode-specific wrapper. `job_test()` becomes a thin call to the new helper plus existing logging/cleanup.

**Tech Stack:** C, Slurm select/cons_tres plugin

---

### Task 1: Add mode helpers for dispatch + deferral

**Files:**
- Modify: `src/plugins/select/cons_tres/job_test.c`

**Step 1: Add mode wrapper for non-WILL_RUN modes**

Add a helper near `_will_run_test()` (after `_will_run_test_with_bitmap()` is fine) to run TEST_ONLY or RUN_NOW without duplicating code:

```c
static int _run_mode_with_bitmap(job_record_t *job_ptr,
				 bitstr_t *node_bitmap,
				 uint32_t min_nodes, uint32_t max_nodes,
				 uint32_t req_nodes, uint16_t mode,
				 uint16_t job_node_req,
				 list_t *preemptee_candidates,
				 list_t **preemptee_job_list,
				 resv_exc_t *resv_exc_ptr)
{
	switch (mode) {
	case SELECT_MODE_TEST_ONLY:
		return _test_only(job_ptr, node_bitmap, min_nodes,
				  max_nodes, req_nodes, job_node_req);
	case SELECT_MODE_RUN_NOW:
		return _run_now(job_ptr, node_bitmap, min_nodes, max_nodes,
				 req_nodes, job_node_req,
				 preemptee_candidates,
				 preemptee_job_list, resv_exc_ptr);
	default:
		return EINVAL;
	}
}
```

**Step 2: Add powered-off deferral dispatcher**

Add a helper that centralizes deferral logic, delegating WILL_RUN directly and running a two-pass deferral for TEST_ONLY/RUN_NOW:

```c
static int _run_mode_with_deferral(job_record_t *job_ptr,
				   bitstr_t *node_bitmap,
				   uint32_t min_nodes, uint32_t max_nodes,
				   uint32_t req_nodes, uint16_t mode,
				   uint16_t job_node_req,
				   list_t *preemptee_candidates,
				   list_t **preemptee_job_list,
				   resv_exc_t *resv_exc_ptr,
				   will_run_data_t *will_run_ptr)
{
	int rc;
	bitstr_t *filtered;

	if (mode == SELECT_MODE_WILL_RUN)
		return _will_run_test(job_ptr, node_bitmap, min_nodes,
				      max_nodes, req_nodes, job_node_req,
				      preemptee_candidates,
				      preemptee_job_list,
				      resv_exc_ptr, will_run_ptr);

	filtered = _filter_powered_on_nodes(node_bitmap);
	if (filtered && bit_set_count(filtered) > 0) {
		rc = _run_mode_with_bitmap(job_ptr, filtered, min_nodes,
					   max_nodes, req_nodes, mode,
					   job_node_req, preemptee_candidates,
					   preemptee_job_list, resv_exc_ptr);
		if (rc == SLURM_SUCCESS) {
			bit_copybits(node_bitmap, filtered);
			FREE_NULL_BITMAP(filtered);
			return rc;
		}
	}

	FREE_NULL_BITMAP(filtered);
	return _run_mode_with_bitmap(job_ptr, node_bitmap, min_nodes,
				     max_nodes, req_nodes, mode,
				     job_node_req, preemptee_candidates,
				     preemptee_job_list, resv_exc_ptr);
}
```

**Step 3: Note test exception**

Tests are skipped per request (no test steps).

**Step 4: Commit**

```bash
git add src/plugins/select/cons_tres/job_test.c
git commit -m "select/cons_tres: centralize mode dispatch deferral"
```

---

### Task 2: Refactor `job_test()` to use new helper

**Files:**
- Modify: `src/plugins/select/cons_tres/job_test.c`

**Step 1: Replace mode branching with helper**

Replace the `if/else` mode dispatch block in `job_test()` with:

```c
rc = _run_mode_with_deferral(job_ptr, node_bitmap, min_nodes,
			    max_nodes, req_nodes, mode, job_node_req,
			    preemptee_candidates, preemptee_job_list,
			    resv_exc_ptr, will_run_ptr);
if (rc == EINVAL) {
	/* Should never get here */
	error("Mode %d is invalid", mode);
	return EINVAL;
}
```

**Step 2: Note test exception**

Tests are skipped per request (no test steps).

**Step 3: Commit**

```bash
git add src/plugins/select/cons_tres/job_test.c
git commit -m "select/cons_tres: simplify job_test mode handling"
```

---

Plan complete and saved to `docs/plans/2026-03-11-cons_tres-mode-dispatch-refactor-plan.md`. Two execution options:

1. Subagent-Driven (this session) - I dispatch fresh subagent per task, review between tasks, fast iteration
2. Parallel Session (separate) - Open new session with executing-plans, batch execution with checkpoints

Which approach?
