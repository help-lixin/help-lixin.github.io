---
layout: post
title: 'MySQL Trace'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1). 前言
> ["本案例的SQL脚本来源于:https://github.com/help-lixin/test_db"](https://github.com/help-lixin/test_db)

### (2). 概述
>  通过trace能详细的分析出,MySQL的执行计划,以及最终选择的索引.

### (3). 开启trace
```
mysql> SHOW VARIABLES LIKE '%optimizer_trace%';
+------------------------------+----------------------------------------------------------------------------+
| Variable_name                | Value                                                                      |
+------------------------------+----------------------------------------------------------------------------+
| optimizer_trace              | enabled=on,one_line=off                                                    |
| optimizer_trace_features     | greedy_search=on,range_optimizer=on,dynamic_range=on,repeated_subselect=on |
| optimizer_trace_limit        | 1                                                                          |
| optimizer_trace_max_mem_size | 16384                                                                      |
| optimizer_trace_offset       | -1                                                                         |
+------------------------------+----------------------------------------------------------------------------+
5 rows in set (0.00 sec)

mysql> SHOW VARIABLES LIKE '%end_markers_in_JSON%';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| end_markers_in_json | ON    |
+---------------------+-------+

mysql> SET SESSION  optimizer_trace="enabled=on",end_markers_in_JSON=on;
Query OK, 0 rows affected (0.00 sec)
```
### (4). trace跟踪
```
mysql> SELECT emp_no  FROM dept_emp WHERE emp_no = 10014;

mysql> select * from information_schema.optimizer_trace \G
*************************** 1. row ***************************
                            QUERY: SELECT emp_no  FROM dept_emp WHERE emp_no = 10014
                            TRACE: {
  "steps": [                      # 阶段
    {
      "join_preparation": {       # 准备阶段
        "select#": 1,
        "steps": [
          {
			  # SQL语句
            "expanded_query": "/* select#1 */ select `dept_emp`.`emp_no` AS `emp_no` from `dept_emp` where (`dept_emp`.`emp_no` = 10014)"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {     # 优化阶段
        "select#": 1,
        "steps": [
          {
            "condition_processing": {           # 查询条件
              "condition": "WHERE",
              "original_condition": "(`dept_emp`.`emp_no` = 10014)",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "multiple equal(10014, `dept_emp`.`emp_no`)"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "multiple equal(10014, `dept_emp`.`emp_no`)"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "multiple equal(10014, `dept_emp`.`emp_no`)"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [               # 表依赖阶段
              {
                "table": "`dept_emp`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`dept_emp`",
                "field": "emp_no",
                "equals": "10014",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`dept_emp`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 331143,
                    "cost": 66968
                  } /* table_scan */,
                  "potential_range_indexes": [         # 可能使用到的索引
                    {
                      "index": "PRIMARY",
                      "usable": true,                  
                      "key_parts": [
                        "emp_no",
                        "dept_no"
                      ] /* key_parts */
                    },
                    {
                      "index": "dept_no",
                      "usable": false,
                      "cause": "not_applicable"
                    }
                  ] /* potential_range_indexes */,
                  "best_covering_index_scan": {
                    "index": "dept_no",
                    "cost": 67360,
                    "chosen": false,
                    "cause": "cost"
                  } /* best_covering_index_scan */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {    # 分析各个索引的成本.
                    "range_scan_alternatives": [
                      {
                        "index": "PRIMARY",
                        "ranges": [
                          "10014 <= emp_no <= 10014"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": false,              
                        "index_only": true,               # 是否使用覆盖索引
                        "rows": 1,                        # 要扫描的行数
                        "cost": 1.21,                     # 扫描这些行数,要花费的时间
                        "chosen": true                    # 是否选择使用这个索引
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"   # 不使用这个索引的原因
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "PRIMARY",
                      "rows": 1,
                      "ranges": [
                        "10014 <= emp_no <= 10014"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,
                    "cost_for_plan": 1.21,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [          # 可以被考虑的执行计划
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`dept_emp`",
                "best_access_path": {               # 最优访问路径
                  "considered_access_paths": [      # MySQL最终会选择这条路径执行
                    {
                      "access_type": "ref",         # scan/全表 ref/引用
                      "index": "PRIMARY",
                      "rows": 1,                    # 要扫描的行数
                      "cost": 1.2,                  # 花费时间
                      "chosen": true                # 选择该路径
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "PRIMARY"
                      } /* range_details */,
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"   # 不使用这个索引的原因
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 1.2,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`dept_emp`.`emp_no` = 10014)",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`dept_emp`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "refine_plan": [
              {
                "table": "`dept_emp`"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```
### (4). 总结
