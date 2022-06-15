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
                  "transformation": "equality_propagation",  # 等值条件句转换
                  "resulting_condition": "multiple equal(10014, `dept_emp`.`emp_no`)"
                },
                {
                  "transformation": "constant_propagation",   # 常量条件句转换
                  "resulting_condition": "multiple equal(10014, `dept_emp`.`emp_no`)"
                },
                {
                  "transformation": "trivial_condition_removal",  # 无效条件移除的转换
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
            "ref_optimizer_key_uses": [    # 列出所有可用的ref类型的索引
              {
                "table": "`dept_emp`",
                "field": "emp_no",
                "equals": "10014",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [          # 估算需要扫描的记录数
              {
                "table": "`dept_emp`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 331143,
                    "cost": 66968
                  } /* table_scan */,
                  "potential_range_indexes": [         # 列出表中所有的索引
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
                  "analyzing_range_alternatives": {    # 分析各个索引的使用成本
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
                        "chosen": true                    # 表示是否使用了该索引
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {    # 分析是否使用了索引合并（index merge）,如果未使用,会在cause中展示原因.如果使用了索引合并,会在该部分展示索引合并的代价
                      "usable": false,
                      "cause": "too_few_roworder_scans" 
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {  # 在summary阶段汇总前一阶段的中间结果确认最后的方案
                    "range_access_plan": {          # range扫描最终选择的执行计划
                      "type": "range_scan",
                      "index": "PRIMARY",
                      "rows": 1,
                      "ranges": [
                        "10014 <= emp_no <= 10014"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,            # 该执行计划的扫描行数
                    "cost_for_plan": 1.21,         # 该执行计划的执行代价
                    "chosen": true                 # 是否选择该执行计划
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [         # 负责对比各可行计划的开销，并选择相对最优的执行计划
              {
                "plan_prefix": [                    # 当前计划的前置执行计划
                ] /* plan_prefix */,
                "table": "`dept_emp`",
                "best_access_path": {               # 最优访问路径
                  "considered_access_paths": [      # MySQL最终会选择这条路径执行
                    {
                      "access_type": "ref",         # 访问类型: scan/全表扫描  ref/引用
                      "index": "PRIMARY",
                      "rows": 1,                    # 要扫描的行数
                      "cost": 1.2,                  # 花费时间
                      "chosen": true                # 确定选择
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "PRIMARY"
                      } /* range_details */,
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"   
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,         # 类似于explain的filtered列，是一个估算值
                "rows_for_plan": 1,                     # 
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
