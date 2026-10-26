**Test gap:** Same issue as `test_get_sales_order_by_id.py`. The unit test `test_get_employee_by_id_missing_id` checks `None` and empty strings, but does not verify the contract between `server.py:124` (`arguments.get("id")`) and the tool handler.

**AC reference:** AC2, criterion 9: "Given `get_employee_by_id` is invoked **without an ID parameter**, When the tool validates input, Then the tool returns `{"error": "Missing required parameter: id"}`."

**Recommendation:** Same as above — verify that `arguments.get("id")` passing `None` (when key is absent) is handled correctly, or add a test that explicitly checks this case.
