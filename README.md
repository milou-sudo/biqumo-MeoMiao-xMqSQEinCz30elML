
最近在南宁出差，搞某个银行的核心系统跑批优化项目。


Oracle 19c Aix 生产环境跑完整体的批要40多分钟左右，在Ob国产环境（国产系统\+国产海光CPU）跑要3个小时😂。这个真不怪Ob，只能说海光处理器真是垃圾中的战斗机。


不过还好，测试环境硬件相对不给力的情况下，给哥优化到95分钟跑完，还算不错成绩了，目标是整体跑批时间搞到80分钟左右，就给领导交差。😁


今天在跑批串行的节点（前后有依赖关系的节点上）遇到条慢SQL，挺有意思的，分享下。


 


**表数据量：**




```
表名                                | 分区规则                                      | 本地索引数量      | 全局索引数量       | 主键PK数量  | 数据行数  
------------------------------------------------------------------------------------------------------------------------------------------------
ENS_CBANK.FFFFFF                    | 非分区表                                      | 0               | 1               | 1          | 72        
ENS_CBANK.XXXXXXX                   | 非分区表                                      | 0               | 1               | 1          | 536       
ENS_CBANK.CCCCCCCC                  | PARTITION BY HASH("INTERNAL_KEY")            | 8               | 2               | 1          | 17289744  
------------------------------------------------------------------------------------------------------------------------------------------------
```


**慢SQL：**




```
SELECT /*+ parallel(16) */ 
    DISTINCT A.PROD_TYPE,
             A.ACCT_NAME,
             A.ACCT_CCY,
             A.SEQ_NO,
             B.BRANCH,
             B.INTERNAL_CLIENT,
             B.COMPANY
FROM (
    SELECT DISTINCT 
            PROD_TYPE, 
            ACCT_NAME, 
            ACCT_CCY, 
            SEQ_NO, 
            REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1, LEVEL) BRANCH_ROLE
      FROM FFFFFF
      CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT(BRANCH_ROLE, '[^|]+')
             AND PRIOR PROD_TYPE = PROD_TYPE
             AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL
      ORDER BY PROD_TYPE
      ) A,
     (
     SELECT DISTINCT 
             BRANCH, 
             INTERNAL_CLIENT, 
             COMPANY, 
             REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1, LEVEL) BRANCH_ROLE
      FROM XXXXXXX
      WHERE AUTO_INNER_FLAG = 'Y'
        AND TRAN_BR_IND = 'Y'
      CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT(BRANCH_ROLE, '[^|]+')
             AND PRIOR BRANCH = BRANCH
             AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL
      ORDER BY BRANCH
      ) B
WHERE A.BRANCH_ROLE = B.BRANCH_ROLE
  AND NOT EXISTS(SELECT 1 FROM CCCCCCCC C WHERE C.BASE_ACCT_NO = (B.BRANCH || A.PROD_TYPE || A.SEQ_NO));
        
20 rows in set (10.027 sec)
```


 


**慢SQL执行计划：**




```
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query Plan                                                                                                                                                                                                                                                                      |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ====================================================================================================                                                                                                                                                                            |
| |ID|OPERATOR                                    |NAME                        |EST.ROWS|EST.TIME(us)|                                                                                                                                                                            |
| ----------------------------------------------------------------------------------------------------                                                                                                                                                                            |
| |0 |HASH DISTINCT                               |                            |2500    |66969       |                                                                                                                                                                            |
| |1 |└─NESTED-LOOP ANTI JOIN                     |                            |2500    |65324       |                                                                                                                                                                            |
| |2 |  ├─HASH JOIN                               |                            |2500    |13287       |                                                                                                                                                                            |
| |3 |  │ ├─SUBPLAN SCAN                          |A                           |91      |1575        |                                                                                                                                                                            |
| |4 |  │ │ └─HASH DISTINCT                       |                            |91      |1574        |                                                                                                                                                                            |
| |5 |  │ │   └─SUBPLAN SCAN                      |VIEW1                       |91      |1513        |                                                                                                                                                                            |
| |6 |  │ │     └─NESTED-LOOP CONNECT BY          |                            |91      |1513        |                                                                                                                                                                            |
| |7 |  │ │       ├─SUBPLAN SCAN                  |VIEW2                       |72      |9           |                                                                                                                                                                            |
| |8 |  │ │       │ └─DISTRIBUTED TABLE FULL SCAN |FFFFFF                      |72      |9           |                                                                                                                                                                            |
| |9 |  │ │       └─SUBPLAN SCAN                  |VIEW3                       |1       |21          |                                                                                                                                                                            |
| |10|  │ │         └─DISTRIBUTED TABLE RANGE SCAN|FFFFFF                      |1       |21          |                                                                                                                                                                            |
| |11|  │ └─SUBPLAN SCAN                          |B                           |313     |11416       |                                                                                                                                                                            |
| |12|  │   └─HASH DISTINCT                       |                            |313     |11415       |                                                                                                                                                                            |
| |13|  │     └─SUBPLAN SCAN                      |VIEW4                       |313     |11236       |                                                                                                                                                                            |
| |14|  │       └─NESTED-LOOP CONNECT BY          |                            |626     |11209       |                                                                                                                                                                            |
| |15|  │         ├─SUBPLAN SCAN                  |VIEW5                       |536     |33          |                                                                                                                                                                            |
| |16|  │         │ └─DISTRIBUTED TABLE FULL SCAN |XXXXXXX                     |536     |32          |                                                                                                                                                                            |
| |17|  │         └─SUBPLAN SCAN                  |VIEW6                       |1       |21          |                                                                                                                                                                            |
| |18|  │           └─DISTRIBUTED TABLE GET       |XXXXXXX                     |1       |21          |                                                                                                                                                                            |
| |19|  └─DISTRIBUTED TABLE RANGE SCAN            |C(IDX_CCCCCCC_GLOBAL_INDEX2)|1       |21          |                                                                                                                                                                            |
| ====================================================================================================                                                                                                                                                                            |
| Outputs & filters:                                                                                                                                                                                                                                                              |
| -------------------------------------                                                                                                                                                                                                                                           |
|   0 - output([A.PROD_TYPE(0x7f6cd80ac480)], [A.ACCT_NAME(0x7f6cd80b4ac0)], [A.ACCT_CCY(0x7f6cd80b8240)], [A.SEQ_NO(0x7f6cd80af970)], [B.BRANCH(0x7f6cd80a6020)],                                                                                                                |
|        [B.INTERNAL_CLIENT(0x7f6cd80bc420)], [B.COMPANY(0x7f6cd80c2b10)]), filter(nil)                                                                                                                                                                                           |
|       distinct([A.PROD_TYPE(0x7f6cd80ac480)], [A.ACCT_NAME(0x7f6cd80b4ac0)], [A.ACCT_CCY(0x7f6cd80b8240)], [A.SEQ_NO(0x7f6cd80af970)], [B.BRANCH(0x7f6cd80a6020)],                                                                                                              |
|        [B.INTERNAL_CLIENT(0x7f6cd80bc420)], [B.COMPANY(0x7f6cd80c2b10)])                                                                                                                                                                                                        |
|   1 - output([A.PROD_TYPE(0x7f6cd80ac480)], [A.ACCT_NAME(0x7f6cd80b4ac0)], [A.ACCT_CCY(0x7f6cd80b8240)], [A.SEQ_NO(0x7f6cd80af970)], [B.BRANCH(0x7f6cd80a6020)],                                                                                                                |
|        [B.INTERNAL_CLIENT(0x7f6cd80bc420)], [B.COMPANY(0x7f6cd80c2b10)]), filter(nil)                                                                                                                                                                                           |
|       conds(nil), nl_params_([B.BRANCH(0x7f6cd80a6020)(:8)], [A.PROD_TYPE(0x7f6cd80ac480)(:9)], [A.SEQ_NO(0x7f6cd80af970)(:10)]), use_batch=false                                                                                                                               |
|   2 - output([A.PROD_TYPE(0x7f6cd80ac480)], [A.ACCT_NAME(0x7f6cd80b4ac0)], [A.ACCT_CCY(0x7f6cd80b8240)], [A.SEQ_NO(0x7f6cd80af970)], [B.BRANCH(0x7f6cd80a6020)],                                                                                                                |
|        [B.INTERNAL_CLIENT(0x7f6cd80bc420)], [B.COMPANY(0x7f6cd80c2b10)]), filter(nil)                                                                                                                                                                                           |
|       equal_conds([A.BRANCH_ROLE(0x7f6cd808b490) = B.BRANCH_ROLE(0x7f6cd808b780)(0x7f6cd808ad40)]), other_conds(nil)                                                                                                                                                            |
|   3 - output([A.BRANCH_ROLE(0x7f6cd808b490)], [A.PROD_TYPE(0x7f6cd80ac480)], [A.SEQ_NO(0x7f6cd80af970)], [A.ACCT_NAME(0x7f6cd80b4ac0)], [A.ACCT_CCY(0x7f6cd80b8240)]), filter(nil)                                                                                              |
|       access([A.BRANCH_ROLE(0x7f6cd808b490)], [A.PROD_TYPE(0x7f6cd80ac480)], [A.SEQ_NO(0x7f6cd80af970)], [A.ACCT_NAME(0x7f6cd80b4ac0)], [A.ACCT_CCY(0x7f6cd80b8240)])                                                                                                           |
|   4 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f6cd80e1a10)], [VIEW1.FFFFFF.ACCT_NAME(0x7f6cd80e2590)], [VIEW1.FFFFFF.ACCT_CCY(0x7f6cd80e1cf0)],                                                                                 |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f6cd80e1fd0)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f6cd80e22b0), VARCHAR2(1048576                                                                                                                |
|       ))(0x7f6cd804f830), cast('[^|]+', VARCHAR2(1048576 ))(0x7f6cd8050380), 1, VIEW1.LEVEL(0x7f6cd80e2870))(0x7f6cd804e000)]), filter(nil)                                                                                                                                     |
|       distinct([VIEW1.FFFFFF.PROD_TYPE(0x7f6cd80e1a10)], [VIEW1.FFFFFF.ACCT_NAME(0x7f6cd80e2590)], [VIEW1.FFFFFF.ACCT_CCY(0x7f6cd80e1cf0)],                                                                               |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f6cd80e1fd0)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f6cd80e22b0), VARCHAR2(1048576                                                                                                                |
|       ))(0x7f6cd804f830), cast('[^|]+', VARCHAR2(1048576 ))(0x7f6cd8050380), 1, VIEW1.LEVEL(0x7f6cd80e2870))(0x7f6cd804e000)])                                                                                                                                                  |
|   5 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f6cd80e1a10)], [VIEW1.FFFFFF.ACCT_CCY(0x7f6cd80e1cf0)], [VIEW1.FFFFFF.SEQ_NO(0x7f6cd80e1fd0)],                                                                                    |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f6cd80e22b0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f6cd80e2590)], [VIEW1.LEVEL(0x7f6cd80e2870)]), filter(nil)                                                                                                    |
|       access([VIEW1.FFFFFF.PROD_TYPE(0x7f6cd80e1a10)], [VIEW1.FFFFFF.ACCT_CCY(0x7f6cd80e1cf0)], [VIEW1.FFFFFF.SEQ_NO(0x7f6cd80e1fd0)],                                                                                    |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f6cd80e22b0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f6cd80e2590)], [VIEW1.LEVEL(0x7f6cd80e2870)])                                                                                                                 |
|   6 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f6cd810a810)], [VIEW3.FFFFFF.ACCT_CCY(0x7f6cd810aaf0)], [VIEW3.FFFFFF.SEQ_NO(0x7f6cd810add0)],                                                                                    |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f6cd810b0b0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f6cd810b390)], [LEVEL(0x7f6cd8042190)]), filter(nil)                                                                                                          |
|       conds(nil), nl_params_([LEVEL(0x7f6cd8042190)(:0)], [VIEW2.FFFFFF.PROD_TYPE(0x7f6cd81096c0)(:1)], [(T_OP_PRIOR, oceanbase.DBMS_RANDOM.VALUE()(0x7f6cd804a480))(0x7f6cd8047a00)(:2                                                                       |
|       )]), use_batch=false                                                                                                                                                                                                                                                      |
|   7 - output([VIEW2.FFFFFF.PROD_TYPE(0x7f6cd81096c0)], [VIEW2.FFFFFF.ACCT_CCY(0x7f6cd81099a0)], [VIEW2.FFFFFF.SEQ_NO(0x7f6cd8109c80)],                                                                                    |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f6cd8109f60)], [VIEW2.FFFFFF.ACCT_NAME(0x7f6cd810a240)]), filter(nil)                                                                                                                                   |
|       access([VIEW2.FFFFFF.PROD_TYPE(0x7f6cd81096c0)], [VIEW2.FFFFFF.ACCT_CCY(0x7f6cd81099a0)], [VIEW2.FFFFFF.SEQ_NO(0x7f6cd8109c80)],                                                                                    |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f6cd8109f60)], [VIEW2.FFFFFF.ACCT_NAME(0x7f6cd810a240)])                                                                                                                                                |
|   8 - output([FFFFFF.PROD_TYPE(0x7f6cd80471e0)], [FFFFFF.ACCT_CCY(0x7f6cd804d700)], [FFFFFF.SEQ_NO(0x7f6cd804dd00)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f6cd8043c50)], [FFFFFF.ACCT_NAME(0x7f6cd804d100)]), filter(nil)                                                                                                                                               |
|       access([FFFFFF.PROD_TYPE(0x7f6cd80471e0)], [FFFFFF.ACCT_CCY(0x7f6cd804d700)], [FFFFFF.SEQ_NO(0x7f6cd804dd00)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f6cd8043c50)], [FFFFFF.ACCT_NAME(0x7f6cd804d100)]), partitions(p0)                                                                                                                                            |
|       is_index_back=false, is_global_index=false,                                                                                                                                                                                                                               |
|       range_key([FFFFFF.PROD_TYPE(0x7f6cd80471e0)], [FFFFFF.ACCT_CCY(0x7f6cd804d700)], [FFFFFF.SEQ_NO(0x7f6cd804dd00)]),                                                                                                  |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true                                                                                                                                                                                                                              |
|   9 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f6cd810a810)], [VIEW3.FFFFFF.ACCT_CCY(0x7f6cd810aaf0)], [VIEW3.FFFFFF.SEQ_NO(0x7f6cd810add0)],                                                                                    |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f6cd810b0b0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f6cd810b390)]), filter(nil), startup_filter([:2                                                                                                               |
|       IS NOT NULL(0x7f56b3552bb0)])                                                                                                                                                                                                                                             |
|       access([VIEW3.FFFFFF.PROD_TYPE(0x7f6cd810a810)], [VIEW3.FFFFFF.ACCT_CCY(0x7f6cd810aaf0)], [VIEW3.FFFFFF.SEQ_NO(0x7f6cd810add0)],                                                                                    |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f6cd810b0b0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f6cd810b390)])                                                                                                                                                |
|  10 - output([FFFFFF.PROD_TYPE(0x7f6cd8108560)], [FFFFFF.ACCT_CCY(0x7f6cd8108840)], [FFFFFF.SEQ_NO(0x7f6cd8108b20)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f6cd8108e00)], [FFFFFF.ACCT_NAME(0x7f6cd81090e0)]), filter([:0 <= REGEXP_COUNT(cast(FFFFFF.BRANCH_ROLE(0x7f6cd8108e00),                                                                     |
|        VARCHAR2(1048576 ))(0x7f56b3553480), cast('[^|]+', VARCHAR2(1048576 ))(0x7f6cd8044b50))(0x7f56b3553d20)(0x7f56b35545c0)])                                                                                                                                                |
|       access([FFFFFF.PROD_TYPE(0x7f6cd8108560)], [FFFFFF.ACCT_CCY(0x7f6cd8108840)], [FFFFFF.SEQ_NO(0x7f6cd8108b20)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f6cd8108e00)], [FFFFFF.ACCT_NAME(0x7f6cd81090e0)]), partitions(p0)                                                                                                                                            |
|       is_index_back=false, is_global_index=false, filter_before_indexback[false],                                                                                                                                                                                               |
|       range_key([FFFFFF.PROD_TYPE(0x7f6cd8108560)], [FFFFFF.ACCT_CCY(0x7f6cd8108840)], [FFFFFF.SEQ_NO(0x7f6cd8108b20)]),                                                                                                  |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true,                                                                                                                                                                                                                             |
|       range_cond([:1 = FFFFFF.PROD_TYPE(0x7f6cd8108560)(0x7f56b3554e80)])                                                                                                                                                                                     |
|  11 - output([B.BRANCH_ROLE(0x7f6cd808b780)], [B.BRANCH(0x7f6cd80a6020)], [B.INTERNAL_CLIENT(0x7f6cd80bc420)], [B.COMPANY(0x7f6cd80c2b10)]), filter(nil)                                                                                                                        |
|       access([B.BRANCH_ROLE(0x7f6cd808b780)], [B.BRANCH(0x7f6cd80a6020)], [B.INTERNAL_CLIENT(0x7f6cd80bc420)], [B.COMPANY(0x7f6cd80c2b10)])                                                                                                                                     |
|  12 - output([VIEW4.XXXXXXX.BRANCH(0x7f6cd8125e10)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8126990)], [VIEW4.XXXXXXX.COMPANY(0x7f6cd8126c70)], [REGEXP_SUBSTR(cast(VIEW4.XXXXXXX.BRANCH_ROLE(0x7f                                                                       |
|       6cd81260f0), VARCHAR2(1048576 ))(0x7f6cd8087290), cast('[^|]+', VARCHAR2(1048576 ))(0x7f6cd8087de0), 1, VIEW4.LEVEL(0x7f6cd8126f50))(0x7f6cd8085740)]), filter(nil)                                                                                                       |
|       distinct([VIEW4.XXXXXXX.BRANCH(0x7f6cd8125e10)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8126990)], [VIEW4.XXXXXXX.COMPANY(0x7f6cd8126c70)], [REGEXP_SUBSTR(cast(VIEW4.XXXXXXX.BRANCH_ROLE(0x                                                                       |
|       7f6cd81260f0), VARCHAR2(1048576 ))(0x7f6cd8087290), cast('[^|]+', VARCHAR2(1048576 ))(0x7f6cd8087de0), 1, VIEW4.LEVEL(0x7f6cd8126f50))(0x7f6cd8085740)])                                                                                                                  |
|  13 - output([VIEW4.XXXXXXX.BRANCH(0x7f6cd8125e10)], [VIEW4.XXXXXXX.BRANCH_ROLE(0x7f6cd81260f0)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8126990)],                                                                                                                        |
|        [VIEW4.XXXXXXX.COMPANY(0x7f6cd8126c70)], [VIEW4.LEVEL(0x7f6cd8126f50)]), filter([VIEW4.XXXXXXX.TRAN_BR_IND(0x7f6cd81266b0) = cast('Y', VARCHAR2(1048576                                                                                                              |
|       ))(0x7f6cd8083cb0)(0x7f6cd807a290)], [VIEW4.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd81263d0) = cast('Y', VARCHAR2(1048576 ))(0x7f6cd80791a0)(0x7f6cd806f780)])                                                                                                                   |
|       access([VIEW4.XXXXXXX.BRANCH(0x7f6cd8125e10)], [VIEW4.XXXXXXX.BRANCH_ROLE(0x7f6cd81260f0)], [VIEW4.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd81263d0)],                                                                                                                        |
|        [VIEW4.XXXXXXX.TRAN_BR_IND(0x7f6cd81266b0)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8126990)], [VIEW4.XXXXXXX.COMPANY(0x7f6cd8126c70)], [VIEW4.LEVEL(0x7f6cd8126f50)])                                                                                              |
|  14 - output([VIEW6.XXXXXXX.BRANCH(0x7f6cd8157b10)], [VIEW6.XXXXXXX.BRANCH_ROLE(0x7f6cd8157df0)], [VIEW6.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd81580d0)],                                                                                                                        |
|        [VIEW6.XXXXXXX.TRAN_BR_IND(0x7f6cd81583b0)], [VIEW6.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8158690)], [VIEW6.XXXXXXX.COMPANY(0x7f6cd8158970)], [LEVEL(0x7f6cd8064af0)]), filter(nil)                                                                                       |
|       conds(nil), nl_params_([LEVEL(0x7f6cd8064af0)(:3)], [VIEW5.XXXXXXX.BRANCH(0x7f6cd81566e0)(:4)], [(T_OP_PRIOR, oceanbase.DBMS_RANDOM.VALUE()(0x7f6cd806cde0))(0x7f6cd806a360)(:5)]),                                                                                     |
|        use_batch=false                                                                                                                                                                                                                                                          |
|  15 - output([VIEW5.XXXXXXX.BRANCH(0x7f6cd81566e0)], [VIEW5.XXXXXXX.BRANCH_ROLE(0x7f6cd81569c0)], [VIEW5.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd8156ca0)],                                                                                                                        |
|        [VIEW5.XXXXXXX.TRAN_BR_IND(0x7f6cd8156f80)], [VIEW5.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8157260)], [VIEW5.XXXXXXX.COMPANY(0x7f6cd8157540)]), filter(nil)                                                                                                                |
|       access([VIEW5.XXXXXXX.BRANCH(0x7f6cd81566e0)], [VIEW5.XXXXXXX.BRANCH_ROLE(0x7f6cd81569c0)], [VIEW5.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd8156ca0)],                                                                                                                        |
|        [VIEW5.XXXXXXX.TRAN_BR_IND(0x7f6cd8156f80)], [VIEW5.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8157260)], [VIEW5.XXXXXXX.COMPANY(0x7f6cd8157540)])                                                                                                                             |
|  16 - output([XXXXXXX.BRANCH(0x7f6cd8069b40)], [XXXXXXX.BRANCH_ROLE(0x7f6cd80665b0)], [XXXXXXX.AUTO_INNER_FLAG(0x7f6cd806fed0)], [XXXXXXX.TRAN_BR_IND(0x7f6cd807a9e0)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f6cd8084e40)], [XXXXXXX.COMPANY(0x7f6cd8085440)]), filter(nil)                                                                                                                                                                           |
|       access([XXXXXXX.BRANCH(0x7f6cd8069b40)], [XXXXXXX.BRANCH_ROLE(0x7f6cd80665b0)], [XXXXXXX.AUTO_INNER_FLAG(0x7f6cd806fed0)], [XXXXXXX.TRAN_BR_IND(0x7f6cd807a9e0)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f6cd8084e40)], [XXXXXXX.COMPANY(0x7f6cd8085440)]), partitions(p0)                                                                                                                                                                        |
|       is_index_back=false, is_global_index=false,                                                                                                                                                                                                                               |
|       range_key([XXXXXXX.BRANCH(0x7f6cd8069b40)]), range(MIN ; MAX)always true                                                                                                                                                                                                |
|  17 - output([VIEW6.XXXXXXX.BRANCH(0x7f6cd8157b10)], [VIEW6.XXXXXXX.BRANCH_ROLE(0x7f6cd8157df0)], [VIEW6.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd81580d0)],                                                                                                                        |
|        [VIEW6.XXXXXXX.TRAN_BR_IND(0x7f6cd81583b0)], [VIEW6.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8158690)], [VIEW6.XXXXXXX.COMPANY(0x7f6cd8158970)]), filter(nil), startup_filter([:5                                                                                            |
|       IS NOT NULL(0x7f556fed9aa0)])                                                                                                                                                                                                                                             |
|       access([VIEW6.XXXXXXX.BRANCH(0x7f6cd8157b10)], [VIEW6.XXXXXXX.BRANCH_ROLE(0x7f6cd8157df0)], [VIEW6.XXXXXXX.AUTO_INNER_FLAG(0x7f6cd81580d0)],                                                                                                                        |
|        [VIEW6.XXXXXXX.TRAN_BR_IND(0x7f6cd81583b0)], [VIEW6.XXXXXXX.INTERNAL_CLIENT(0x7f6cd8158690)], [VIEW6.XXXXXXX.COMPANY(0x7f6cd8158970)])                                                                                                                             |
|  18 - output([XXXXXXX.BRANCH(0x7f6cd814cee0)], [XXXXXXX.BRANCH_ROLE(0x7f6cd814d1c0)], [XXXXXXX.AUTO_INNER_FLAG(0x7f6cd814d4a0)], [XXXXXXX.TRAN_BR_IND(0x7f6cd8151960)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f6cd8155e20)], [XXXXXXX.COMPANY(0x7f6cd8156100)]), filter([:3 <= REGEXP_COUNT(cast(XXXXXXX.BRANCH_ROLE(0x7f6cd814d1c0),                                                                                                                |
|        VARCHAR2(1048576 ))(0x7f556feda370), cast('[^|]+', VARCHAR2(1048576 ))(0x7f6cd80674b0))(0x7f556fedac10)(0x7f556fedb4b0)])                                                                                                                                                |
|       access([XXXXXXX.BRANCH(0x7f6cd814cee0)], [XXXXXXX.BRANCH_ROLE(0x7f6cd814d1c0)], [XXXXXXX.AUTO_INNER_FLAG(0x7f6cd814d4a0)], [XXXXXXX.TRAN_BR_IND(0x7f6cd8151960)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f6cd8155e20)], [XXXXXXX.COMPANY(0x7f6cd8156100)]), partitions(p0)                                                                                                                                                                        |
|       is_index_back=false, is_global_index=false, filter_before_indexback[false],                                                                                                                                                                                               |
|       range_key([XXXXXXX.BRANCH(0x7f6cd814cee0)]), range(MIN ; MAX)always true,                                                                                                                                                                                               |
|       range_cond([:4 = XXXXXXX.BRANCH(0x7f6cd814cee0)(0x7f556fedbd70)])                                                                                                                                                                                                       |
|  19 - output(nil), filter(nil)                                                                                                                                                                                                                                                  |
|       access(nil), partitions(p0)                                                                                                                                                                                                                                               |
|       is_index_back=false, is_global_index=true,                                                                                                                                                                                                                                |
|       range_key([C.BASE_ACCT_NO(0x7f6cd80a5d30)], [C.INTERNAL_KEY(0x7f6cd80a3600)]), range(MIN ; MAX),                                                                                                                                                                          |
|       range_cond([C.BASE_ACCT_NO(0x7f6cd80a5d30) = (T_OP_CNN, (T_OP_CNN, :8, :9)(0x7f556ff63dc0), :10)(0x7f556ff648d0)(0x7f556ff653e0)])                                                                                                                                        |
| Used Hint:                                                                                                                                                                                                                                                                      |
| -------------------------------------                                                                                                                                                                                                                                           |
|   /*+                                                                                                                                                                                                                                                                           |
|                                                                                                                                                                                                                                                                                 |
|       PARALLEL(16)                                                                                                                                                                                                                                                              |
|   */                                                                                                                                                                                                                                                                            |
| Qb name trace:                                                                                                                                                                                                                                                                  |
| -------------------------------------                                                                                                                                                                                                                                           |
|   stmt_id:0, stmt_type:T_EXPLAIN                                                                                                                                                                                                                                                |
|   stmt_id:1, SEL$1 > SEL$C1AAFB47 > SEL$D45D735D > SEL$FF87951C                                                                                                                                                                                                                 |
|   stmt_id:2, SEL$2                                                                                                                                                                                                                                                              |
|   stmt_id:3, SEL$3                                                                                                                                                                                                                                                              |
|   stmt_id:4, SEL$4                                                                                                                                                                                                                                                              |
|   stmt_id:5, parent:SEL$2  > SEL$9B6BAA9A                                                                                                                                                                                                                                       |
|   stmt_id:6, parent:SEL$2  > SEL$9B6BAA9B                                                                                                                                                                                                                                       |
|   stmt_id:7, parent:SEL$9B6BAA9B  > SEL$E382C6D8_1                                                                                                                                                                                                                              |
|   stmt_id:8, parent:SEL$3  > SEL$B648BD05                                                                                                                                                                                                                                       |
|   stmt_id:9, parent:SEL$3  > SEL$B648BD06                                                                                                                                                                                                                                       |
|   stmt_id:10, parent:SEL$B648BD06  > SEL$8174842E_1                                                                                                                                                                                                                             |
| Outline Data:                                                                                                                                                                                                                                                                   |
| -------------------------------------                                                                                                                                                                                                                                           |
|   /*+                                                                                                                                                                                                                                                                           |
|       BEGIN_OUTLINE_DATA                                                                                                                                                                                                                                                        |
|       USE_HASH_DISTINCT(@"SEL$FF87951C")                                                                                                                                                                                                                                        |
|       LEADING(@"SEL$FF87951C" (("A"@"SEL$1" "B"@"SEL$1") "ENS_CBANK"."C"@"SEL$4"))                                                                                                                                                                                              |
|       USE_NL(@"SEL$FF87951C" "ENS_CBANK"."C"@"SEL$4")                                                                                                                                                                                                                           |
|       USE_HASH(@"SEL$FF87951C" "B"@"SEL$1")                                                                                                                                                                                                                                     |
|       USE_HASH_DISTINCT(@"SEL$2")                                                                                                                                                                                                                                               |
|       LEADING(@"SEL$9B6BAA9A" ("VIEW2"@"SEL$9B6BAA9A" "VIEW3"@"SEL$9B6BAA9A"))                                                                                                                                                                                                  |
|       USE_NL(@"SEL$9B6BAA9A" "VIEW3"@"SEL$9B6BAA9A")                                                                                                                                                                                                                            |
|       FULL(@"SEL$9B6BAA9B" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                      |
|       USE_DAS(@"SEL$9B6BAA9B" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                   |
|       FULL(@"SEL$E382C6D8_1" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                    |
|       USE_DAS(@"SEL$E382C6D8_1" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                 |
|       USE_HASH_DISTINCT(@"SEL$3")                                                                                                                                                                                                                                               |
|       LEADING(@"SEL$B648BD05" ("VIEW5"@"SEL$B648BD05" "VIEW6"@"SEL$B648BD05"))                                                                                                                                                                                                  |
|       USE_NL(@"SEL$B648BD05" "VIEW6"@"SEL$B648BD05")                                                                                                                                                                                                                            |
|       FULL(@"SEL$B648BD06" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                     |
|       USE_DAS(@"SEL$B648BD06" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                  |
|       FULL(@"SEL$8174842E_1" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                   |
|       USE_DAS(@"SEL$8174842E_1" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                |
|       INDEX(@"SEL$FF87951C" "C"@"SEL$4" "IDX_CCCCCCC_GLOBAL_INDEX2")                                                                                                                                                                                                            |
|       USE_DAS(@"SEL$FF87951C" "C"@"SEL$4")                                                                                                                                                                                                                                      |
|       SIMPLIFY_ORDER_BY(@"SEL$1")                                                                                                                                                                                                                                               |
|       UNNEST(@"SEL$4")                                                                                                                                                                                                                                                          |
|       MERGE(@"SEL$4" > "SEL$D45D735D")                                                                                                                                                                                                                                          |
|       OPTIMIZER_FEATURES_ENABLE('4.2.1.0')                                                                                                                                                                                                                                      |
|       END_OUTLINE_DATA                                                                                                                                                                                                                                                          |
|   */                                                                                                                                                                                                                                                                            |
| Optimization Info:                                                                                                                                                                                                                                                              |
| -------------------------------------                                                                                                                                                                                                                                           |
|   FFFFFF:                                                                                                                                                                                                                                                     |
|       table_rows:72                                                                                                                                                                                                                                                             |
|       physical_range_rows:72                                                                                                                                                                                                                                                    |
|       logical_range_rows:72                                                                                                                                                                                                                                                     |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:72                                                                                                                                                                                                                                                            |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                                                                                            |
|       stats version:1722575467958265                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   FFFFFF:                                                                                                                                                                                                                                                     |
|       table_rows:72                                                                                                                                                                                                                                                             |
|       physical_range_rows:1                                                                                                                                                                                                                                                     |
|       logical_range_rows:1                                                                                                                                                                                                                                                      |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:0                                                                                                                                                                                                                                                             |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                                                                                            |
|       stats version:1722575467958265                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   XXXXXXX:                                                                                                                                                                                                                                                                    |
|       table_rows:536                                                                                                                                                                                                                                                            |
|       physical_range_rows:536                                                                                                                                                                                                                                                   |
|       logical_range_rows:536                                                                                                                                                                                                                                                    |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:536                                                                                                                                                                                                                                                           |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[XXXXXXX]                                                                                                                                                                                                                                           |
|       stats version:1722575463271440                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   XXXXXXX:                                                                                                                                                                                                                                                                    |
|       table_rows:536                                                                                                                                                                                                                                                            |
|       physical_range_rows:1                                                                                                                                                                                                                                                     |
|       logical_range_rows:1                                                                                                                                                                                                                                                      |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:0                                                                                                                                                                                                                                                             |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[XXXXXXX]                                                                                                                                                                                                                                           |
|       stats version:1722575463271440                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   C:                                                                                                                                                                                                                                                                            |
|       table_rows:17289678                                                                                                                                                                                                                                                       |
|       physical_range_rows:2                                                                                                                                                                                                                                                     |
|       logical_range_rows:2                                                                                                                                                                                                                                                      |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:2                                                                                                                                                                                                                                                             |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[IDX_CCCCCCC_GLOBAL_INDEX2, IDX_CCCCCCCC_GLOBAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX1, IDX_CCCCCCCC_LOCAL_INDEX2, IDX_CCCCCCCC_LOCAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX4, IDX_CCCCCCCC_LOCAL_INDEX5, IDX_CCCCCCCC_LOCAL_INDEX6, IDX_CCCCCCCC_LOCAL_INDEX7, CCCCCCCC] |
|       pruned_index_name:[IDX_CCCCCCCC_GLOBAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX1, IDX_CCCCCCCC_LOCAL_INDEX2, IDX_CCCCCCCC_LOCAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX4, IDX_CCCCCCCC_LOCAL_INDEX5, IDX_CCCCCCCC_LOCAL_INDEX6, IDX_CCCCCCCC_LOCAL_INDEX7, CCCCCCCC]                              |
|       stats version:1728378528448689                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   Plan Type:                                                                                                                                                                                                                                                                    |
|       LOCAL                                                                                                                                                                                                                                                                     |
|   Note:                                                                                                                                                                                                                                                                         |
|       Degree of Parallelisim is 1 because stmt contain pl_udf which force das scan                                                                                                                                                                                              |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
231 rows in set (0.028 sec)
```


 


**主要慢的SQL和计划：**




```
-- 慢的SQL：
SELECT DISTINCT 
        PROD_TYPE, 
        ACCT_NAME, 
        ACCT_CCY, 
        SEQ_NO,
        REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1 , LEVEL) BRANCH_ROLE
    FROM FFFFFF 
        CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+') 
        AND PRIOR PROD_TYPE = PROD_TYPE
        AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL ORDER BY PROD_TYPE
323 rows in set (10.204 sec)


+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query Plan                                                                                                                                                                                                  |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ==========================================================================================                                                                                                                  |
| |ID|OPERATOR                              |NAME                    |EST.ROWS|EST.TIME(us)|                                                                                                                  |
| ------------------------------------------------------------------------------------------                                                                                                                  |
| |0 |MERGE DISTINCT                        |                        |91      |1580        |                                                                                                                  |
| |1 |└─SORT                                |                        |91      |1579        |                                                                                                                  |
| |2 |  └─SUBPLAN SCAN                      |VIEW1                   |91      |1513        |                                                                                                                  |
| |3 |    └─NESTED-LOOP CONNECT BY          |                        |91      |1513        |                                                                                                                  |
| |4 |      ├─SUBPLAN SCAN                  |VIEW2                   |72      |9           |                                                                                                                  |
| |5 |      │ └─DISTRIBUTED TABLE FULL SCAN |FFFFFF                  |72      |9           |                                                                                                                  |
| |6 |      └─SUBPLAN SCAN                  |VIEW3                   |1       |21          |                                                                                                                  |
| |7 |        └─DISTRIBUTED TABLE RANGE SCAN|FFFFFF                  |1       |21          |                                                                                                                  |
| ==========================================================================================                                                                                                                  |
| Outputs & filters:                                                                                                                                                                                          |
| -------------------------------------                                                                                                                                                                       |
|   0 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f7f40648d50)], [VIEW1.FFFFFF.ACCT_NAME(0x7f7f406498d0)], [VIEW1.FFFFFF.ACCT_CCY(0x7f7f40649030)],             |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f7f40649310)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f7f406495f0), VARCHAR2(1048576                                            |
|       ))(0x7f7f40631730), cast('[^|]+', VARCHAR2(1048576 ))(0x7f7f40632280), 1, VIEW1.LEVEL(0x7f7f40649bb0))(0x7f7f4062ff00)]), filter(nil)                                                                 |
|       distinct([VIEW1.FFFFFF.PROD_TYPE(0x7f7f40648d50)], [VIEW1.FFFFFF.ACCT_NAME(0x7f7f406498d0)], [VIEW1.FFFFFF.ACCT_CCY(0x7f7f40649030)],           |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f7f40649310)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f7f406495f0), VARCHAR2(1048576                                            |
|       ))(0x7f7f40631730), cast('[^|]+', VARCHAR2(1048576 ))(0x7f7f40632280), 1, VIEW1.LEVEL(0x7f7f40649bb0))(0x7f7f4062ff00)])                                                                              |
|   1 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f7f40648d50)], [VIEW1.FFFFFF.ACCT_NAME(0x7f7f406498d0)], [VIEW1.FFFFFF.ACCT_CCY(0x7f7f40649030)],             |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f7f40649310)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f7f406495f0), VARCHAR2(1048576                                            |
|       ))(0x7f7f40631730), cast('[^|]+', VARCHAR2(1048576 ))(0x7f7f40632280), 1, VIEW1.LEVEL(0x7f7f40649bb0))(0x7f7f4062ff00)]), filter(nil)                                                                 |
|       sort_keys([VIEW1.FFFFFF.PROD_TYPE(0x7f7f40648d50), ASC], [VIEW1.FFFFFF.ACCT_NAME(0x7f7f406498d0), ASC], [VIEW1.FFFFFF.ACCT_CCY(0x7f7f40649030   |
|       ), ASC], [VIEW1.FFFFFF.SEQ_NO(0x7f7f40649310), ASC], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f7f406495f0), VARCHAR2(1048576                               |
|       ))(0x7f7f40631730), cast('[^|]+', VARCHAR2(1048576 ))(0x7f7f40632280), 1, VIEW1.LEVEL(0x7f7f40649bb0))(0x7f7f4062ff00), ASC])                                                                         |
|   2 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f7f40648d50)], [VIEW1.FFFFFF.ACCT_CCY(0x7f7f40649030)], [VIEW1.FFFFFF.SEQ_NO(0x7f7f40649310)],                |
|        [VIEW1.FFFFFF.ACCT_NAME(0x7f7f406498d0)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f7f406495f0), VARCHAR2(1048576                                         |
|       ))(0x7f7f40631730), cast('[^|]+', VARCHAR2(1048576 ))(0x7f7f40632280), 1, VIEW1.LEVEL(0x7f7f40649bb0))(0x7f7f4062ff00)]), filter(nil)                                                                 |
|       access([VIEW1.FFFFFF.PROD_TYPE(0x7f7f40648d50)], [VIEW1.FFFFFF.ACCT_CCY(0x7f7f40649030)], [VIEW1.FFFFFF.SEQ_NO(0x7f7f40649310)],                |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f7f406495f0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f7f406498d0)], [VIEW1.LEVEL(0x7f7f40649bb0)])                                             |
|   3 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f7f40671b50)], [VIEW3.FFFFFF.ACCT_CCY(0x7f7f40671e30)], [VIEW3.FFFFFF.SEQ_NO(0x7f7f40672110)],                |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f7f406723f0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f7f406726d0)], [LEVEL(0x7f7f40624090)]), filter(nil)                                      |
|       conds(nil), nl_params_([LEVEL(0x7f7f40624090)(:0)], [VIEW2.FFFFFF.PROD_TYPE(0x7f7f40670a00)(:1)], [(T_OP_PRIOR, oceanbase.DBMS_RANDOM.VALUE()(0x7f7f4062c380))(0x7f7f40629900)(:2   |
|       )]), use_batch=false                                                                                                                                                                                  |
|   4 - output([VIEW2.FFFFFF.PROD_TYPE(0x7f7f40670a00)], [VIEW2.FFFFFF.ACCT_CCY(0x7f7f40670ce0)], [VIEW2.FFFFFF.SEQ_NO(0x7f7f40670fc0)],                |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f7f406712a0)], [VIEW2.FFFFFF.ACCT_NAME(0x7f7f40671580)]), filter(nil)                                                               |
|       access([VIEW2.FFFFFF.PROD_TYPE(0x7f7f40670a00)], [VIEW2.FFFFFF.ACCT_CCY(0x7f7f40670ce0)], [VIEW2.FFFFFF.SEQ_NO(0x7f7f40670fc0)],                |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f7f406712a0)], [VIEW2.FFFFFF.ACCT_NAME(0x7f7f40671580)])                                                                            |
|   5 - output([FFFFFF.PROD_TYPE(0x7f7f406290e0)], [FFFFFF.ACCT_CCY(0x7f7f4062f600)], [FFFFFF.SEQ_NO(0x7f7f4062fc00)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f7f40625b50)], [FFFFFF.ACCT_NAME(0x7f7f4062f000)]), filter(nil)                                                                           |
|       access([FFFFFF.PROD_TYPE(0x7f7f406290e0)], [FFFFFF.ACCT_CCY(0x7f7f4062f600)], [FFFFFF.SEQ_NO(0x7f7f4062fc00)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f7f40625b50)], [FFFFFF.ACCT_NAME(0x7f7f4062f000)]), partitions(p0)                                                                        |
|       is_index_back=false, is_global_index=false,                                                                                                                                                           |
|       range_key([FFFFFF.PROD_TYPE(0x7f7f406290e0)], [FFFFFF.ACCT_CCY(0x7f7f4062f600)], [FFFFFF.SEQ_NO(0x7f7f4062fc00)]),                              |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true                                                                                                                                                          |
|   6 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f7f40671b50)], [VIEW3.FFFFFF.ACCT_CCY(0x7f7f40671e30)], [VIEW3.FFFFFF.SEQ_NO(0x7f7f40672110)],                |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f7f406723f0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f7f406726d0)]), filter(nil), startup_filter([:2                                           |
|       IS NOT NULL(0x7f7f407b6c70)])                                                                                                                                                                         |
|       access([VIEW3.FFFFFF.PROD_TYPE(0x7f7f40671b50)], [VIEW3.FFFFFF.ACCT_CCY(0x7f7f40671e30)], [VIEW3.FFFFFF.SEQ_NO(0x7f7f40672110)],                |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f7f406723f0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f7f406726d0)])                                                                            |
|   7 - output([FFFFFF.PROD_TYPE(0x7f7f4066f8a0)], [FFFFFF.ACCT_CCY(0x7f7f4066fb80)], [FFFFFF.SEQ_NO(0x7f7f4066fe60)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f7f40670140)], [FFFFFF.ACCT_NAME(0x7f7f40670420)]), filter([:0 <= REGEXP_COUNT(cast(FFFFFF.BRANCH_ROLE(0x7f7f40670140), |
|        VARCHAR2(1048576 ))(0x7f7f407b7540), cast('[^|]+', VARCHAR2(1048576 ))(0x7f7f40626a50))(0x7f7f407b7de0)(0x7f7f407b8680)])                                                                            |
|       access([FFFFFF.PROD_TYPE(0x7f7f4066f8a0)], [FFFFFF.ACCT_CCY(0x7f7f4066fb80)], [FFFFFF.SEQ_NO(0x7f7f4066fe60)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f7f40670140)], [FFFFFF.ACCT_NAME(0x7f7f40670420)]), partitions(p0)                                                                        |
|       is_index_back=false, is_global_index=false, filter_before_indexback[false],                                                                                                                           |
|       range_key([FFFFFF.PROD_TYPE(0x7f7f4066f8a0)], [FFFFFF.ACCT_CCY(0x7f7f4066fb80)], [FFFFFF.SEQ_NO(0x7f7f4066fe60)]),                              |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true,                                                                                                                                                         |
|       range_cond([:1 = FFFFFF.PROD_TYPE(0x7f7f4066f8a0)(0x7f7f407b8f40)])                                                                                                                 |
| Used Hint:                                                                                                                                                                                                  |
| -------------------------------------                                                                                                                                                                       |
|   /*+                                                                                                                                                                                                       |
|                                                                                                                                                                                                             |
|   */                                                                                                                                                                                                        |
| Qb name trace:                                                                                                                                                                                              |
| -------------------------------------                                                                                                                                                                       |
|   stmt_id:0, stmt_type:T_EXPLAIN                                                                                                                                                                            |
|   stmt_id:1, SEL$1                                                                                                                                                                                          |
|   stmt_id:2, parent:SEL$1  > SEL$C07C92B2                                                                                                                                                                   |
|   stmt_id:3, parent:SEL$1  > SEL$C07C92B3                                                                                                                                                                   |
|   stmt_id:4, parent:SEL$C07C92B3  > SEL$4DA317D2_1                                                                                                                                                          |
| Outline Data:                                                                                                                                                                                               |
| -------------------------------------                                                                                                                                                                       |
|   /*+                                                                                                                                                                                                       |
|       BEGIN_OUTLINE_DATA                                                                                                                                                                                    |
|       LEADING(@"SEL$C07C92B2" ("VIEW2"@"SEL$C07C92B2" "VIEW3"@"SEL$C07C92B2"))                                                                                                                              |
|       USE_NL(@"SEL$C07C92B2" "VIEW3"@"SEL$C07C92B2")                                                                                                                                                        |
|       FULL(@"SEL$C07C92B3" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                                  |
|       USE_DAS(@"SEL$C07C92B3" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                               |
|       FULL(@"SEL$4DA317D2_1" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                                |
|       USE_DAS(@"SEL$4DA317D2_1" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                             |
|       OPTIMIZER_FEATURES_ENABLE('4.2.1.0')                                                                                                                                                                  |
|       END_OUTLINE_DATA                                                                                                                                                                                      |
|   */                                                                                                                                                                                                        |
| Optimization Info:                                                                                                                                                                                          |
| -------------------------------------                                                                                                                                                                       |
|   FFFFFF:                                                                                                                                                                                 |
|       table_rows:72                                                                                                                                                                                         |
|       physical_range_rows:72                                                                                                                                                                                |
|       logical_range_rows:72                                                                                                                                                                                 |
|       index_back_rows:0                                                                                                                                                                                     |
|       output_rows:72                                                                                                                                                                                        |
|       table_dop:1                                                                                                                                                                                           |
|       dop_method:DAS DOP                                                                                                                                                                                    |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                        |
|       stats version:1722575467958265                                                                                                                                                                        |
|       dynamic sampling level:0                                                                                                                                                                              |
|   FFFFFF:                                                                                                                                                                                 |
|       table_rows:72                                                                                                                                                                                         |
|       physical_range_rows:1                                                                                                                                                                                 |
|       logical_range_rows:1                                                                                                                                                                                  |
|       index_back_rows:0                                                                                                                                                                                     |
|       output_rows:0                                                                                                                                                                                         |
|       table_dop:1                                                                                                                                                                                           |
|       dop_method:DAS DOP                                                                                                                                                                                    |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                        |
|       stats version:1722575467958265                                                                                                                                                                        |
|       dynamic sampling level:0                                                                                                                                                                              |
|   Plan Type:                                                                                                                                                                                                |
|       LOCAL                                                                                                                                                                                                 |
|   Note:                                                                                                                                                                                                     |
|       Degree of Parallelisim is 1 because stmt contain pl_udf which force das scan                                                                                                                          |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
113 rows in set (0.012 sec)
```


这条SQL执行时间跑10秒，慢在 PRIOR DBMS\_RANDOM.VALUE IS NOT NULL ORDER BY PROD\_TYPE 这段，如果去掉就非常快，毫秒级别出结果。


但是去掉以后返回的结果集数量和没去掉之前差别很大，BRANCH\_ROLE 字段存储的内容是 'Role1\|Role2\|Role3\|Role4\|Role5' 这种数据格式，行的数据存放到一个列里面。


 


因为这条SQL没写 START WITH 条件, 我猜开发想实现的逻辑是将父节点和子节点关联成功的前提下，将每行的 BRANCH\_ROLE 字段内  'Role1\|Role2\|Role3\|Role4\|Role5'  数据拆分成每一行**（REGEXP\_SUBSTR(BRANCH\_ROLE, '\[^\|]\+', 1 , LEVEL) BRANCH\_ROLE）**，才写了个 PRIOR DBMS\_RANDOM.VALUE IS NOT NULL 条件，将不确定性带入递归条件中。


只要达到 CONNECT BY NOCYCLE LEVEL \<\= REGEXP\_COUNT (BRANCH\_ROLE, '\[^\|]\+')  这个计数器条件以后，就停止递归。




```
例如：

　　PROD_TYPE 　BRANCH_ROLE

　　　　1　　　　　Role1|Role2|Role3|Role4|Role5

拆成：

　　PROD_TYPE 　BRANCH_ROLE

　　　　1　　　　　Role1

　　　　1　　　　　Role2

　　　　1　　　　　Role3

　　　　1　　　　  Role4

　　　　1　　　　　Role5
```


假如 PROD\_TYPE 有 10行，每行列拆行的情况下，数据会翻倍。


 


**不过开发这个逻辑没写全，我按照他这个逻辑改写一版。**


**改写SQL（PROD\_TYPE 、ACCT\_CCY 、SEQ\_NO联合主键）：**




```
-- 改写：
    SELECT 
        DISTINCT PROD_TYPE, 
                 ACCT_NAME, 
                 ACCT_CCY, 
                 SEQ_NO, 
                 REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1 , LEVEL) BRANCH_ROLE
        FROM FFFFFF 
            CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+') 
            AND PRIOR PROD_TYPE = PROD_TYPE 
            AND PRIOR ACCT_CCY = ACCT_CCY 
            AND PRIOR SEQ_NO = SEQ_NO 
            AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL 


323 rows in set (0.058 sec)



+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query Plan                                                                                                                                                                                                  |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| =======================================================================================                                                                                                                     |
| |ID|OPERATOR                           |NAME                    |EST.ROWS|EST.TIME(us)|                                                                                                                     |
| ---------------------------------------------------------------------------------------                                                                                                                     |
| |0 |HASH DISTINCT                      |                        |85      |1567        |                                                                                                                     |
| |1 |└─SUBPLAN SCAN                     |VIEW1                   |85      |1511        |                                                                                                                     |
| |2 |  └─NESTED-LOOP CONNECT BY         |                        |85      |1510        |                                                                                                                     |
| |3 |    ├─SUBPLAN SCAN                 |VIEW2                   |72      |9           |                                                                                                                     |
| |4 |    │ └─DISTRIBUTED TABLE FULL SCAN|FFFFFF                  |72      |9           |                                                                                                                     |
| |5 |    └─SUBPLAN SCAN                 |VIEW3                   |1       |21          |                                                                                                                     |
| |6 |      └─DISTRIBUTED TABLE GET      |FFFFFF                  |1       |21          |                                                                                                                     |
| =======================================================================================                                                                                                                     |
| Outputs & filters:                                                                                                                                                                                          |
| -------------------------------------                                                                                                                                                                       |
|   0 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0c4cd10)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0c4d890)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0c4cff0)],             |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f59f0c4d2d0)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0c4d5b0), VARCHAR2(1048576                                            |
|       ))(0x7f59f0c351b0), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c35d00), 1, VIEW1.LEVEL(0x7f59f0c4db70))(0x7f59f0c33c90)]), filter(nil)                                                                 |
|       distinct([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0c4cd10)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0c4d890)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0c4cff0)],           |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f59f0c4d2d0)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0c4d5b0), VARCHAR2(1048576                                            |
|       ))(0x7f59f0c351b0), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c35d00), 1, VIEW1.LEVEL(0x7f59f0c4db70))(0x7f59f0c33c90)])                                                                              |
|   1 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0c4cd10)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0c4cff0)], [VIEW1.FFFFFF.SEQ_NO(0x7f59f0c4d2d0)],                |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0c4d5b0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0c4d890)], [VIEW1.LEVEL(0x7f59f0c4db70)]), filter(nil)                                |
|       access([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0c4cd10)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0c4cff0)], [VIEW1.FFFFFF.SEQ_NO(0x7f59f0c4d2d0)],                |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0c4d5b0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0c4d890)], [VIEW1.LEVEL(0x7f59f0c4db70)])                                             |
|   2 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f59f0c75d50)], [VIEW3.FFFFFF.ACCT_CCY(0x7f59f0c76030)], [VIEW3.FFFFFF.SEQ_NO(0x7f59f0c76310)],                |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f59f0c765f0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f59f0c768d0)], [LEVEL(0x7f59f0c24ac0)]), filter(nil)                                      |
|       conds(nil), nl_params_([LEVEL(0x7f59f0c24ac0)(:0)], [VIEW2.FFFFFF.PROD_TYPE(0x7f59f0c74c00)(:1)], [VIEW2.FFFFFF.ACCT_CCY(0x7f59f0c74ee0)(:2)],                    |
|        [VIEW2.FFFFFF.SEQ_NO(0x7f59f0c751c0)(:3)], [(T_OP_PRIOR, oceanbase.DBMS_RANDOM.VALUE()(0x7f59f0c306d0))(0x7f59f0c2dc50)(:4)]), use_batch=false                                     |
|   3 - output([VIEW2.FFFFFF.PROD_TYPE(0x7f59f0c74c00)], [VIEW2.FFFFFF.ACCT_CCY(0x7f59f0c74ee0)], [VIEW2.FFFFFF.SEQ_NO(0x7f59f0c751c0)],                |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f59f0c754a0)], [VIEW2.FFFFFF.ACCT_NAME(0x7f59f0c75780)]), filter(nil)                                                               |
|       access([VIEW2.FFFFFF.PROD_TYPE(0x7f59f0c74c00)], [VIEW2.FFFFFF.ACCT_CCY(0x7f59f0c74ee0)], [VIEW2.FFFFFF.SEQ_NO(0x7f59f0c751c0)],                |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f59f0c754a0)], [VIEW2.FFFFFF.ACCT_NAME(0x7f59f0c75780)])                                                                            |
|   4 - output([FFFFFF.PROD_TYPE(0x7f59f0c29b10)], [FFFFFF.ACCT_CCY(0x7f59f0c2b7a0)], [FFFFFF.SEQ_NO(0x7f59f0c2d430)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0c26580)], [FFFFFF.ACCT_NAME(0x7f59f0c33350)]), filter(nil)                                                                           |
|       access([FFFFFF.PROD_TYPE(0x7f59f0c29b10)], [FFFFFF.ACCT_CCY(0x7f59f0c2b7a0)], [FFFFFF.SEQ_NO(0x7f59f0c2d430)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0c26580)], [FFFFFF.ACCT_NAME(0x7f59f0c33350)]), partitions(p0)                                                                        |
|       is_index_back=false, is_global_index=false,                                                                                                                                                           |
|       range_key([FFFFFF.PROD_TYPE(0x7f59f0c29b10)], [FFFFFF.ACCT_CCY(0x7f59f0c2b7a0)], [FFFFFF.SEQ_NO(0x7f59f0c2d430)]),                              |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true                                                                                                                                                          |
|   5 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f59f0c75d50)], [VIEW3.FFFFFF.ACCT_CCY(0x7f59f0c76030)], [VIEW3.FFFFFF.SEQ_NO(0x7f59f0c76310)],                |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f59f0c765f0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f59f0c768d0)]), filter(nil), startup_filter([:4                                           |
|       IS NOT NULL(0x7f59f0dc46b0)])                                                                                                                                                                         |
|       access([VIEW3.FFFFFF.PROD_TYPE(0x7f59f0c75d50)], [VIEW3.FFFFFF.ACCT_CCY(0x7f59f0c76030)], [VIEW3.FFFFFF.SEQ_NO(0x7f59f0c76310)],                |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f59f0c765f0)], [VIEW3.FFFFFF.ACCT_NAME(0x7f59f0c768d0)])                                                                            |
|   6 - output([FFFFFF.PROD_TYPE(0x7f59f0c73aa0)], [FFFFFF.ACCT_CCY(0x7f59f0c73d80)], [FFFFFF.SEQ_NO(0x7f59f0c74060)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0c74340)], [FFFFFF.ACCT_NAME(0x7f59f0c74620)]), filter([:0 <= REGEXP_COUNT(cast(FFFFFF.BRANCH_ROLE(0x7f59f0c74340), |
|        VARCHAR2(1048576 ))(0x7f59f0dc4f80), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c27480))(0x7f59f0dc5820)(0x7f59f0dc60c0)])                                                                            |
|       access([FFFFFF.PROD_TYPE(0x7f59f0c73aa0)], [FFFFFF.ACCT_CCY(0x7f59f0c73d80)], [FFFFFF.SEQ_NO(0x7f59f0c74060)],                                  |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0c74340)], [FFFFFF.ACCT_NAME(0x7f59f0c74620)]), partitions(p0)                                                                        |
|       is_index_back=false, is_global_index=false, filter_before_indexback[false],                                                                                                                           |
|       range_key([FFFFFF.PROD_TYPE(0x7f59f0c73aa0)], [FFFFFF.ACCT_CCY(0x7f59f0c73d80)], [FFFFFF.SEQ_NO(0x7f59f0c74060)]),                              |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true,                                                                                                                                                         |
|       range_cond([:1 = FFFFFF.PROD_TYPE(0x7f59f0c73aa0)(0x7f59f0dc6980)], [:2 = FFFFFF.ACCT_CCY(0x7f59f0c73d80)(0x7f59f0dc7210)],                                       |
|        [:3 = FFFFFF.SEQ_NO(0x7f59f0c74060)(0x7f59f0dc7aa0)])                                                                                                                              |
| Used Hint:                                                                                                                                                                                                  |
| -------------------------------------                                                                                                                                                                       |
|   /*+                                                                                                                                                                                                       |
|                                                                                                                                                                                                             |
|   */                                                                                                                                                                                                        |
| Qb name trace:                                                                                                                                                                                              |
| -------------------------------------                                                                                                                                                                       |
|   stmt_id:0, stmt_type:T_EXPLAIN                                                                                                                                                                            |
|   stmt_id:1, SEL$1                                                                                                                                                                                          |
|   stmt_id:2, parent:SEL$1  > SEL$C07C92B2                                                                                                                                                                   |
|   stmt_id:3, parent:SEL$1  > SEL$C07C92B3                                                                                                                                                                   |
|   stmt_id:4, parent:SEL$C07C92B3  > SEL$4DA317D2_1                                                                                                                                                          |
| Outline Data:                                                                                                                                                                                               |
| -------------------------------------                                                                                                                                                                       |
|   /*+                                                                                                                                                                                                       |
|       BEGIN_OUTLINE_DATA                                                                                                                                                                                    |
|       USE_HASH_DISTINCT(@"SEL$1")                                                                                                                                                                           |
|       LEADING(@"SEL$C07C92B2" ("VIEW2"@"SEL$C07C92B2" "VIEW3"@"SEL$C07C92B2"))                                                                                                                              |
|       USE_NL(@"SEL$C07C92B2" "VIEW3"@"SEL$C07C92B2")                                                                                                                                                        |
|       FULL(@"SEL$C07C92B3" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                                  |
|       USE_DAS(@"SEL$C07C92B3" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                               |
|       FULL(@"SEL$4DA317D2_1" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                                |
|       USE_DAS(@"SEL$4DA317D2_1" "ENS_CBANK"."FFFFFF"@"SEL$1")                                                                                                                             |
|       OPTIMIZER_FEATURES_ENABLE('4.2.1.0')                                                                                                                                                                  |
|       END_OUTLINE_DATA                                                                                                                                                                                      |
|   */                                                                                                                                                                                                        |
| Optimization Info:                                                                                                                                                                                          |
| -------------------------------------                                                                                                                                                                       |
|   FFFFFF:                                                                                                                                                                                 |
|       table_rows:72                                                                                                                                                                                         |
|       physical_range_rows:72                                                                                                                                                                                |
|       logical_range_rows:72                                                                                                                                                                                 |
|       index_back_rows:0                                                                                                                                                                                     |
|       output_rows:72                                                                                                                                                                                        |
|       table_dop:1                                                                                                                                                                                           |
|       dop_method:DAS DOP                                                                                                                                                                                    |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                        |
|       stats version:1722575467958265                                                                                                                                                                        |
|       dynamic sampling level:0                                                                                                                                                                              |
|   FFFFFF:                                                                                                                                                                                 |
|       table_rows:72                                                                                                                                                                                         |
|       physical_range_rows:1                                                                                                                                                                                 |
|       logical_range_rows:1                                                                                                                                                                                  |
|       index_back_rows:0                                                                                                                                                                                     |
|       output_rows:0                                                                                                                                                                                         |
|       table_dop:1                                                                                                                                                                                           |
|       dop_method:DAS DOP                                                                                                                                                                                    |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                        |
|       stats version:1722575467958265                                                                                                                                                                        |
|       dynamic sampling level:0                                                                                                                                                                              |
|   Plan Type:                                                                                                                                                                                                |
|       LOCAL                                                                                                                                                                                                 |
|   Note:                                                                                                                                                                                                     |
|       Degree of Parallelisim is 1 because stmt contain pl_udf which force das scan                                                                                                                          |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
107 rows in set (0.016 sec)
```


简单点说就是每一行数据展开 BRANCH\_ROLE 列内的数据。改写的SQL看得懂就看，看不懂也不多说，说起来又一大堆😁。


改成这样后SQL执行时间降到 58毫秒，优化也算是结束了， XXXXXXX 那段递归不慢不需要优化，因为 BRANCH 是唯一键。


 


**整体SQL速度和执行计划：**




```
SELECT 
    DISTINCT A.PROD_TYPE, 
             A.ACCT_NAME, 
             A.ACCT_CCY, 
             A.SEQ_NO, 
             B.BRANCH, 
             B.INTERNAL_CLIENT, 
             B.COMPANY
        FROM
        ( 
    SELECT 
        DISTINCT PROD_TYPE, 
                 ACCT_NAME, 
                 ACCT_CCY, 
                 SEQ_NO, 
                 REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1 , LEVEL) BRANCH_ROLE
        FROM FFFFFF 
            CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+') 
            AND PRIOR PROD_TYPE = PROD_TYPE 
            AND PRIOR ACCT_CCY = ACCT_CCY 
            AND PRIOR SEQ_NO = SEQ_NO 
            AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL 
            ) A ,
        ( 
        SELECT 
            DISTINCT BRANCH, 
                     INTERNAL_CLIENT, 
                     COMPANY, 
                     REGEXP_SUBSTR (BRANCH_ROLE, '[^|]+', 1, LEVEL ) BRANCH_ROLE
        FROM XXXXXXX 
            WHERE AUTO_INNER_FLAG = 'Y' 
                AND TRAN_BR_IND = 'Y' 
                CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+')
                AND PRIOR BRANCH = BRANCH 
                AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL 
        ) B
        WHERE A.BRANCH_ROLE = B.BRANCH_ROLE AND NOT EXISTS ( SELECT 1 FROM CCCCCCCC C WHERE C.BASE_ACCT_NO = ( B.BRANCH || A.PROD_TYPE || A.SEQ_NO ))
        
        
        
        
20 rows in set (0.556 sec)

+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query Plan                                                                                                                                                                                                                                                                      |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ===================================================================================================                                                                                                                                                                             |
| |ID|OPERATOR                                   |NAME                        |EST.ROWS|EST.TIME(us)|                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------                                                                                                                                                                             |
| |0 |HASH DISTINCT                              |                            |2323    |63136       |                                                                                                                                                                             |
| |1 |└─NESTED-LOOP ANTI JOIN                    |                            |2323    |61608       |                                                                                                                                                                             |
| |2 |  ├─HASH JOIN                              |                            |2323    |13264       |                                                                                                                                                                             |
| |3 |  │ ├─SUBPLAN SCAN                         |A                           |85      |1568        |                                                                                                                                                                             |
| |4 |  │ │ └─HASH DISTINCT                      |                            |85      |1567        |                                                                                                                                                                             |
| |5 |  │ │   └─SUBPLAN SCAN                     |VIEW1                       |85      |1511        |                                                                                                                                                                             |
| |6 |  │ │     └─NESTED-LOOP CONNECT BY         |                            |85      |1510        |                                                                                                                                                                             |
| |7 |  │ │       ├─SUBPLAN SCAN                 |VIEW2                       |72      |9           |                                                                                                                                                                             |
| |8 |  │ │       │ └─DISTRIBUTED TABLE FULL SCAN|FFFFFF                      |72      |9           |                                                                                                                                                                             |
| |9 |  │ │       └─SUBPLAN SCAN                 |VIEW3                       |1       |21          |                                                                                                                                                                             |
| |10|  │ │         └─DISTRIBUTED TABLE GET      |FFFFFF                      |1       |21          |                                                                                                                                                                             |
| |11|  │ └─SUBPLAN SCAN                         |B                           |313     |11416       |                                                                                                                                                                             |
| |12|  │   └─HASH DISTINCT                      |                            |313     |11415       |                                                                                                                                                                             |
| |13|  │     └─SUBPLAN SCAN                     |VIEW4                       |313     |11236       |                                                                                                                                                                             |
| |14|  │       └─NESTED-LOOP CONNECT BY         |                            |626     |11209       |                                                                                                                                                                             |
| |15|  │         ├─SUBPLAN SCAN                 |VIEW5                       |536     |33          |                                                                                                                                                                             |
| |16|  │         │ └─DISTRIBUTED TABLE FULL SCAN|XXXXXXX                     |536     |32          |                                                                                                                                                                             |
| |17|  │         └─SUBPLAN SCAN                 |VIEW6                       |1       |21          |                                                                                                                                                                             |
| |18|  │           └─DISTRIBUTED TABLE GET      |XXXXXXX                     |1       |21          |                                                                                                                                                                             |
| |19|  └─DISTRIBUTED TABLE RANGE SCAN           |C(IDX_CCCCCCC_GLOBAL_INDEX2)|1       |21          |                                                                                                                                                                             |
| ===================================================================================================                                                                                                                                                                             |
| Outputs & filters:                                                                                                                                                                                                                                                              |
| -------------------------------------                                                                                                                                                                                                                                           |
|   0 - output([A.PROD_TYPE(0x7f59f0caf5e0)], [A.ACCT_NAME(0x7f59f0cb7c20)], [A.ACCT_CCY(0x7f59f0cbb3a0)], [A.SEQ_NO(0x7f59f0cb2ad0)], [B.BRANCH(0x7f59f0ca9180)],                                                                                                                |
|        [B.INTERNAL_CLIENT(0x7f59f0cbf580)], [B.COMPANY(0x7f59f0cc5c70)]), filter(nil)                                                                                                                                                                                           |
|       distinct([A.PROD_TYPE(0x7f59f0caf5e0)], [A.ACCT_NAME(0x7f59f0cb7c20)], [A.ACCT_CCY(0x7f59f0cbb3a0)], [A.SEQ_NO(0x7f59f0cb2ad0)], [B.BRANCH(0x7f59f0ca9180)],                                                                                                              |
|        [B.INTERNAL_CLIENT(0x7f59f0cbf580)], [B.COMPANY(0x7f59f0cc5c70)])                                                                                                                                                                                                        |
|   1 - output([A.PROD_TYPE(0x7f59f0caf5e0)], [A.ACCT_NAME(0x7f59f0cb7c20)], [A.ACCT_CCY(0x7f59f0cbb3a0)], [A.SEQ_NO(0x7f59f0cb2ad0)], [B.BRANCH(0x7f59f0ca9180)],                                                                                                                |
|        [B.INTERNAL_CLIENT(0x7f59f0cbf580)], [B.COMPANY(0x7f59f0cc5c70)]), filter(nil)                                                                                                                                                                                           |
|       conds(nil), nl_params_([B.BRANCH(0x7f59f0ca9180)(:10)], [A.PROD_TYPE(0x7f59f0caf5e0)(:11)], [A.SEQ_NO(0x7f59f0cb2ad0)(:12)]), use_batch=false                                                                                                                             |
|   2 - output([A.PROD_TYPE(0x7f59f0caf5e0)], [A.ACCT_NAME(0x7f59f0cb7c20)], [A.ACCT_CCY(0x7f59f0cbb3a0)], [A.SEQ_NO(0x7f59f0cb2ad0)], [B.BRANCH(0x7f59f0ca9180)],                                                                                                                |
|        [B.INTERNAL_CLIENT(0x7f59f0cbf580)], [B.COMPANY(0x7f59f0cc5c70)]), filter(nil)                                                                                                                                                                                           |
|       equal_conds([A.BRANCH_ROLE(0x7f59f0c8e5f0) = B.BRANCH_ROLE(0x7f59f0c8e8e0)(0x7f59f0c8dea0)]), other_conds(nil)                                                                                                                                                            |
|   3 - output([A.BRANCH_ROLE(0x7f59f0c8e5f0)], [A.PROD_TYPE(0x7f59f0caf5e0)], [A.SEQ_NO(0x7f59f0cb2ad0)], [A.ACCT_NAME(0x7f59f0cb7c20)], [A.ACCT_CCY(0x7f59f0cbb3a0)]), filter(nil)                                                                                              |
|       access([A.BRANCH_ROLE(0x7f59f0c8e5f0)], [A.PROD_TYPE(0x7f59f0caf5e0)], [A.SEQ_NO(0x7f59f0cb2ad0)], [A.ACCT_NAME(0x7f59f0cb7c20)], [A.ACCT_CCY(0x7f59f0cbb3a0)])                                                                                                           |
|   4 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0ce50b0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0ce5c30)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0ce5390)],                                                                                 |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f59f0ce5670)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0ce5950), VARCHAR2(1048576                                                                                                                |
|       ))(0x7f59f0c52a60), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c535b0), 1, VIEW1.LEVEL(0x7f59f0ce5f10))(0x7f59f0c51540)]), filter(nil)                                                                                                                                     |
|       distinct([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0ce50b0)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0ce5c30)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0ce5390)],                                                                               |
|        [VIEW1.FFFFFF.SEQ_NO(0x7f59f0ce5670)], [REGEXP_SUBSTR(cast(VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0ce5950), VARCHAR2(1048576                                                                                                                |
|       ))(0x7f59f0c52a60), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c535b0), 1, VIEW1.LEVEL(0x7f59f0ce5f10))(0x7f59f0c51540)])                                                                                                                                                  |
|   5 - output([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0ce50b0)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0ce5390)], [VIEW1.FFFFFF.SEQ_NO(0x7f59f0ce5670)],                                                                                    |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0ce5950)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0ce5c30)], [VIEW1.LEVEL(0x7f59f0ce5f10)]), filter(nil)                                                                                                    |
|       access([VIEW1.FFFFFF.PROD_TYPE(0x7f59f0ce50b0)], [VIEW1.FFFFFF.ACCT_CCY(0x7f59f0ce5390)], [VIEW1.FFFFFF.SEQ_NO(0x7f59f0ce5670)],                                                                                    |
|        [VIEW1.FFFFFF.BRANCH_ROLE(0x7f59f0ce5950)], [VIEW1.FFFFFF.ACCT_NAME(0x7f59f0ce5c30)], [VIEW1.LEVEL(0x7f59f0ce5f10)])                                                                                                                 |
|   6 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f59f0d0e0f0)], [VIEW3.FFFFFF.ACCT_CCY(0x7f59f0d0e3d0)], [VIEW3.FFFFFF.SEQ_NO(0x7f59f0d0e6b0)],                                                                                    |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f59f0d0e990)], [VIEW3.FFFFFF.ACCT_NAME(0x7f59f0d0ec70)], [LEVEL(0x7f59f0c42370)]), filter(nil)                                                                                                          |
|       conds(nil), nl_params_([LEVEL(0x7f59f0c42370)(:0)], [VIEW2.FFFFFF.PROD_TYPE(0x7f59f0d0cfa0)(:1)], [VIEW2.FFFFFF.ACCT_CCY(0x7f59f0d0d280)(:2)],                                                                                        |
|        [VIEW2.FFFFFF.SEQ_NO(0x7f59f0d0d560)(:3)], [(T_OP_PRIOR, oceanbase.DBMS_RANDOM.VALUE()(0x7f59f0c4df80))(0x7f59f0c4b500)(:4)]), use_batch=false                                                                                                         |
|   7 - output([VIEW2.FFFFFF.PROD_TYPE(0x7f59f0d0cfa0)], [VIEW2.FFFFFF.ACCT_CCY(0x7f59f0d0d280)], [VIEW2.FFFFFF.SEQ_NO(0x7f59f0d0d560)],                                                                                    |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f59f0d0d840)], [VIEW2.FFFFFF.ACCT_NAME(0x7f59f0d0db20)]), filter(nil)                                                                                                                                   |
|       access([VIEW2.FFFFFF.PROD_TYPE(0x7f59f0d0cfa0)], [VIEW2.FFFFFF.ACCT_CCY(0x7f59f0d0d280)], [VIEW2.FFFFFF.SEQ_NO(0x7f59f0d0d560)],                                                                                    |
|        [VIEW2.FFFFFF.BRANCH_ROLE(0x7f59f0d0d840)], [VIEW2.FFFFFF.ACCT_NAME(0x7f59f0d0db20)])                                                                                                                                                |
|   8 - output([FFFFFF.PROD_TYPE(0x7f59f0c473c0)], [FFFFFF.ACCT_CCY(0x7f59f0c49050)], [FFFFFF.SEQ_NO(0x7f59f0c4ace0)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0c43e30)], [FFFFFF.ACCT_NAME(0x7f59f0c50c00)]), filter(nil)                                                                                                                                               |
|       access([FFFFFF.PROD_TYPE(0x7f59f0c473c0)], [FFFFFF.ACCT_CCY(0x7f59f0c49050)], [FFFFFF.SEQ_NO(0x7f59f0c4ace0)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0c43e30)], [FFFFFF.ACCT_NAME(0x7f59f0c50c00)]), partitions(p0)                                                                                                                                            |
|       is_index_back=false, is_global_index=false,                                                                                                                                                                                                                               |
|       range_key([FFFFFF.PROD_TYPE(0x7f59f0c473c0)], [FFFFFF.ACCT_CCY(0x7f59f0c49050)], [FFFFFF.SEQ_NO(0x7f59f0c4ace0)]),                                                                                                  |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true                                                                                                                                                                                                                              |
|   9 - output([VIEW3.FFFFFF.PROD_TYPE(0x7f59f0d0e0f0)], [VIEW3.FFFFFF.ACCT_CCY(0x7f59f0d0e3d0)], [VIEW3.FFFFFF.SEQ_NO(0x7f59f0d0e6b0)],                                                                                    |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f59f0d0e990)], [VIEW3.FFFFFF.ACCT_NAME(0x7f59f0d0ec70)]), filter(nil), startup_filter([:4                                                                                                               |
|       IS NOT NULL(0x7f6250d5bf70)])                                                                                                                                                                                                                                             |
|       access([VIEW3.FFFFFF.PROD_TYPE(0x7f59f0d0e0f0)], [VIEW3.FFFFFF.ACCT_CCY(0x7f59f0d0e3d0)], [VIEW3.FFFFFF.SEQ_NO(0x7f59f0d0e6b0)],                                                                                    |
|        [VIEW3.FFFFFF.BRANCH_ROLE(0x7f59f0d0e990)], [VIEW3.FFFFFF.ACCT_NAME(0x7f59f0d0ec70)])                                                                                                                                                |
|  10 - output([FFFFFF.PROD_TYPE(0x7f59f0d0be40)], [FFFFFF.ACCT_CCY(0x7f59f0d0c120)], [FFFFFF.SEQ_NO(0x7f59f0d0c400)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0d0c6e0)], [FFFFFF.ACCT_NAME(0x7f59f0d0c9c0)]), filter([:0 <= REGEXP_COUNT(cast(FFFFFF.BRANCH_ROLE(0x7f59f0d0c6e0),                                                                     |
|        VARCHAR2(1048576 ))(0x7f6250d5c840), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c44d30))(0x7f6250d5d0e0)(0x7f6250d5d980)])                                                                                                                                                |
|       access([FFFFFF.PROD_TYPE(0x7f59f0d0be40)], [FFFFFF.ACCT_CCY(0x7f59f0d0c120)], [FFFFFF.SEQ_NO(0x7f59f0d0c400)],                                                                                                      |
|        [FFFFFF.BRANCH_ROLE(0x7f59f0d0c6e0)], [FFFFFF.ACCT_NAME(0x7f59f0d0c9c0)]), partitions(p0)                                                                                                                                            |
|       is_index_back=false, is_global_index=false, filter_before_indexback[false],                                                                                                                                                                                               |
|       range_key([FFFFFF.PROD_TYPE(0x7f59f0d0be40)], [FFFFFF.ACCT_CCY(0x7f59f0d0c120)], [FFFFFF.SEQ_NO(0x7f59f0d0c400)]),                                                                                                  |
|        range(MIN,MIN,MIN ; MAX,MAX,MAX)always true,                                                                                                                                                                                                                             |
|       range_cond([:1 = FFFFFF.PROD_TYPE(0x7f59f0d0be40)(0x7f6250d5e240)], [:2 = FFFFFF.ACCT_CCY(0x7f59f0d0c120)(0x7f6250d5ead0)],                                                                                                           |
|        [:3 = FFFFFF.SEQ_NO(0x7f59f0d0c400)(0x7f6250d5f360)])                                                                                                                                                                                                  |
|  11 - output([B.BRANCH_ROLE(0x7f59f0c8e8e0)], [B.BRANCH(0x7f59f0ca9180)], [B.INTERNAL_CLIENT(0x7f59f0cbf580)], [B.COMPANY(0x7f59f0cc5c70)]), filter(nil)                                                                                                                        |
|       access([B.BRANCH_ROLE(0x7f59f0c8e8e0)], [B.BRANCH(0x7f59f0ca9180)], [B.INTERNAL_CLIENT(0x7f59f0cbf580)], [B.COMPANY(0x7f59f0cc5c70)])                                                                                                                                     |
|  12 - output([VIEW4.XXXXXXX.BRANCH(0x7f59f0d29b70)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d2a6f0)], [VIEW4.XXXXXXX.COMPANY(0x7f59f0d2a9d0)], [REGEXP_SUBSTR(cast(VIEW4.XXXXXXX.BRANCH_ROLE(0x7f                                                                       |
|       59f0d29e50), VARCHAR2(1048576 ))(0x7f59f0c8a3f0), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c8af40), 1, VIEW4.LEVEL(0x7f59f0d2acb0))(0x7f59f0c88bb0)]), filter(nil)                                                                                                       |
|       distinct([VIEW4.XXXXXXX.BRANCH(0x7f59f0d29b70)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d2a6f0)], [VIEW4.XXXXXXX.COMPANY(0x7f59f0d2a9d0)], [REGEXP_SUBSTR(cast(VIEW4.XXXXXXX.BRANCH_ROLE(0x                                                                       |
|       7f59f0d29e50), VARCHAR2(1048576 ))(0x7f59f0c8a3f0), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c8af40), 1, VIEW4.LEVEL(0x7f59f0d2acb0))(0x7f59f0c88bb0)])                                                                                                                  |
|  13 - output([VIEW4.XXXXXXX.BRANCH(0x7f59f0d29b70)], [VIEW4.XXXXXXX.BRANCH_ROLE(0x7f59f0d29e50)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d2a6f0)],                                                                                                                        |
|        [VIEW4.XXXXXXX.COMPANY(0x7f59f0d2a9d0)], [VIEW4.LEVEL(0x7f59f0d2acb0)]), filter([VIEW4.XXXXXXX.TRAN_BR_IND(0x7f59f0d2a410) = cast('Y', VARCHAR2(1048576                                                                                                              |
|       ))(0x7f59f0c87120)(0x7f59f0c7d700)], [VIEW4.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d2a130) = cast('Y', VARCHAR2(1048576 ))(0x7f59f0c7c610)(0x7f59f0c72bf0)])                                                                                                                   |
|       access([VIEW4.XXXXXXX.BRANCH(0x7f59f0d29b70)], [VIEW4.XXXXXXX.BRANCH_ROLE(0x7f59f0d29e50)], [VIEW4.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d2a130)],                                                                                                                        |
|        [VIEW4.XXXXXXX.TRAN_BR_IND(0x7f59f0d2a410)], [VIEW4.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d2a6f0)], [VIEW4.XXXXXXX.COMPANY(0x7f59f0d2a9d0)], [VIEW4.LEVEL(0x7f59f0d2acb0)])                                                                                              |
|  14 - output([VIEW6.XXXXXXX.BRANCH(0x7f59f0d5b870)], [VIEW6.XXXXXXX.BRANCH_ROLE(0x7f59f0d5bb50)], [VIEW6.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d5be30)],                                                                                                                        |
|        [VIEW6.XXXXXXX.TRAN_BR_IND(0x7f59f0d5c110)], [VIEW6.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d5c3f0)], [VIEW6.XXXXXXX.COMPANY(0x7f59f0d5c6d0)], [LEVEL(0x7f59f0c67f60)]), filter(nil)                                                                                       |
|       conds(nil), nl_params_([LEVEL(0x7f59f0c67f60)(:5)], [VIEW5.XXXXXXX.BRANCH(0x7f59f0d5a440)(:6)], [(T_OP_PRIOR, oceanbase.DBMS_RANDOM.VALUE()(0x7f59f0c70250))(0x7f59f0c6d7d0)(:7)]),                                                                                     |
|        use_batch=false                                                                                                                                                                                                                                                          |
|  15 - output([VIEW5.XXXXXXX.BRANCH(0x7f59f0d5a440)], [VIEW5.XXXXXXX.BRANCH_ROLE(0x7f59f0d5a720)], [VIEW5.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d5aa00)],                                                                                                                        |
|        [VIEW5.XXXXXXX.TRAN_BR_IND(0x7f59f0d5ace0)], [VIEW5.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d5afc0)], [VIEW5.XXXXXXX.COMPANY(0x7f59f0d5b2a0)]), filter(nil)                                                                                                                |
|       access([VIEW5.XXXXXXX.BRANCH(0x7f59f0d5a440)], [VIEW5.XXXXXXX.BRANCH_ROLE(0x7f59f0d5a720)], [VIEW5.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d5aa00)],                                                                                                                        |
|        [VIEW5.XXXXXXX.TRAN_BR_IND(0x7f59f0d5ace0)], [VIEW5.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d5afc0)], [VIEW5.XXXXXXX.COMPANY(0x7f59f0d5b2a0)])                                                                                                                             |
|  16 - output([XXXXXXX.BRANCH(0x7f59f0c6cfb0)], [XXXXXXX.BRANCH_ROLE(0x7f59f0c69a20)], [XXXXXXX.AUTO_INNER_FLAG(0x7f59f0c73340)], [XXXXXXX.TRAN_BR_IND(0x7f59f0c7de50)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f59f0c882b0)], [XXXXXXX.COMPANY(0x7f59f0c888b0)]), filter(nil)                                                                                                                                                                           |
|       access([XXXXXXX.BRANCH(0x7f59f0c6cfb0)], [XXXXXXX.BRANCH_ROLE(0x7f59f0c69a20)], [XXXXXXX.AUTO_INNER_FLAG(0x7f59f0c73340)], [XXXXXXX.TRAN_BR_IND(0x7f59f0c7de50)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f59f0c882b0)], [XXXXXXX.COMPANY(0x7f59f0c888b0)]), partitions(p0)                                                                                                                                                                        |
|       is_index_back=false, is_global_index=false,                                                                                                                                                                                                                               |
|       range_key([XXXXXXX.BRANCH(0x7f59f0c6cfb0)]), range(MIN ; MAX)always true                                                                                                                                                                                                |
|  17 - output([VIEW6.XXXXXXX.BRANCH(0x7f59f0d5b870)], [VIEW6.XXXXXXX.BRANCH_ROLE(0x7f59f0d5bb50)], [VIEW6.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d5be30)],                                                                                                                        |
|        [VIEW6.XXXXXXX.TRAN_BR_IND(0x7f59f0d5c110)], [VIEW6.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d5c3f0)], [VIEW6.XXXXXXX.COMPANY(0x7f59f0d5c6d0)]), filter(nil), startup_filter([:7                                                                                            |
|       IS NOT NULL(0x7f6442ad9aa0)])                                                                                                                                                                                                                                             |
|       access([VIEW6.XXXXXXX.BRANCH(0x7f59f0d5b870)], [VIEW6.XXXXXXX.BRANCH_ROLE(0x7f59f0d5bb50)], [VIEW6.XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d5be30)],                                                                                                                        |
|        [VIEW6.XXXXXXX.TRAN_BR_IND(0x7f59f0d5c110)], [VIEW6.XXXXXXX.INTERNAL_CLIENT(0x7f59f0d5c3f0)], [VIEW6.XXXXXXX.COMPANY(0x7f59f0d5c6d0)])                                                                                                                             |
|  18 - output([XXXXXXX.BRANCH(0x7f59f0d50c40)], [XXXXXXX.BRANCH_ROLE(0x7f59f0d50f20)], [XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d51200)], [XXXXXXX.TRAN_BR_IND(0x7f59f0d556c0)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f59f0d59b80)], [XXXXXXX.COMPANY(0x7f59f0d59e60)]), filter([:5 <= REGEXP_COUNT(cast(XXXXXXX.BRANCH_ROLE(0x7f59f0d50f20),                                                                                                                |
|        VARCHAR2(1048576 ))(0x7f6442ada370), cast('[^|]+', VARCHAR2(1048576 ))(0x7f59f0c6a920))(0x7f6442adac10)(0x7f6442adb4b0)])                                                                                                                                                |
|       access([XXXXXXX.BRANCH(0x7f59f0d50c40)], [XXXXXXX.BRANCH_ROLE(0x7f59f0d50f20)], [XXXXXXX.AUTO_INNER_FLAG(0x7f59f0d51200)], [XXXXXXX.TRAN_BR_IND(0x7f59f0d556c0)],                                                                                                 |
|        [XXXXXXX.INTERNAL_CLIENT(0x7f59f0d59b80)], [XXXXXXX.COMPANY(0x7f59f0d59e60)]), partitions(p0)                                                                                                                                                                        |
|       is_index_back=false, is_global_index=false, filter_before_indexback[false],                                                                                                                                                                                               |
|       range_key([XXXXXXX.BRANCH(0x7f59f0d50c40)]), range(MIN ; MAX)always true,                                                                                                                                                                                               |
|       range_cond([:6 = XXXXXXX.BRANCH(0x7f59f0d50c40)(0x7f6442adbd70)])                                                                                                                                                                                                       |
|  19 - output(nil), filter(nil)                                                                                                                                                                                                                                                  |
|       access(nil), partitions(p0)                                                                                                                                                                                                                                               |
|       is_index_back=false, is_global_index=true,                                                                                                                                                                                                                                |
|       range_key([C.BASE_ACCT_NO(0x7f59f0ca8e90)], [C.INTERNAL_KEY(0x7f59f0ca6760)]), range(MIN ; MAX),                                                                                                                                                                          |
|       range_cond([C.BASE_ACCT_NO(0x7f59f0ca8e90) = (T_OP_CNN, (T_OP_CNN, :10, :11)(0x7f6442b63dc0), :12)(0x7f6442b648d0)(0x7f6442b653e0)])                                                                                                                                      |
| Used Hint:                                                                                                                                                                                                                                                                      |
| -------------------------------------                                                                                                                                                                                                                                           |
|   /*+                                                                                                                                                                                                                                                                           |
|                                                                                                                                                                                                                                                                                 |
|   */                                                                                                                                                                                                                                                                            |
| Qb name trace:                                                                                                                                                                                                                                                                  |
| -------------------------------------                                                                                                                                                                                                                                           |
|   stmt_id:0, stmt_type:T_EXPLAIN                                                                                                                                                                                                                                                |
|   stmt_id:1, SEL$1 > SEL$4A3BEEAA > SEL$B09914F6                                                                                                                                                                                                                                |
|   stmt_id:2, SEL$2                                                                                                                                                                                                                                                              |
|   stmt_id:3, SEL$3                                                                                                                                                                                                                                                              |
|   stmt_id:4, SEL$4                                                                                                                                                                                                                                                              |
|   stmt_id:5, parent:SEL$2  > SEL$9B6BAA9A                                                                                                                                                                                                                                       |
|   stmt_id:6, parent:SEL$2  > SEL$9B6BAA9B                                                                                                                                                                                                                                       |
|   stmt_id:7, parent:SEL$9B6BAA9B  > SEL$E382C6D8_1                                                                                                                                                                                                                              |
|   stmt_id:8, parent:SEL$3  > SEL$B648BD05                                                                                                                                                                                                                                       |
|   stmt_id:9, parent:SEL$3  > SEL$B648BD06                                                                                                                                                                                                                                       |
|   stmt_id:10, parent:SEL$B648BD06  > SEL$8174842E_1                                                                                                                                                                                                                             |
| Outline Data:                                                                                                                                                                                                                                                                   |
| -------------------------------------                                                                                                                                                                                                                                           |
|   /*+                                                                                                                                                                                                                                                                           |
|       BEGIN_OUTLINE_DATA                                                                                                                                                                                                                                                        |
|       USE_HASH_DISTINCT(@"SEL$B09914F6")                                                                                                                                                                                                                                        |
|       LEADING(@"SEL$B09914F6" (("A"@"SEL$1" "B"@"SEL$1") "ENS_CBANK"."C"@"SEL$4"))                                                                                                                                                                                              |
|       USE_NL(@"SEL$B09914F6" "ENS_CBANK"."C"@"SEL$4")                                                                                                                                                                                                                           |
|       USE_HASH(@"SEL$B09914F6" "B"@"SEL$1")                                                                                                                                                                                                                                     |
|       USE_HASH_DISTINCT(@"SEL$2")                                                                                                                                                                                                                                               |
|       LEADING(@"SEL$9B6BAA9A" ("VIEW2"@"SEL$9B6BAA9A" "VIEW3"@"SEL$9B6BAA9A"))                                                                                                                                                                                                  |
|       USE_NL(@"SEL$9B6BAA9A" "VIEW3"@"SEL$9B6BAA9A")                                                                                                                                                                                                                            |
|       FULL(@"SEL$9B6BAA9B" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                      |
|       USE_DAS(@"SEL$9B6BAA9B" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                   |
|       FULL(@"SEL$E382C6D8_1" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                    |
|       USE_DAS(@"SEL$E382C6D8_1" "ENS_CBANK"."FFFFFF"@"SEL$2")                                                                                                                                                                                                 |
|       USE_HASH_DISTINCT(@"SEL$3")                                                                                                                                                                                                                                               |
|       LEADING(@"SEL$B648BD05" ("VIEW5"@"SEL$B648BD05" "VIEW6"@"SEL$B648BD05"))                                                                                                                                                                                                  |
|       USE_NL(@"SEL$B648BD05" "VIEW6"@"SEL$B648BD05")                                                                                                                                                                                                                            |
|       FULL(@"SEL$B648BD06" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                     |
|       USE_DAS(@"SEL$B648BD06" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                  |
|       FULL(@"SEL$8174842E_1" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                   |
|       USE_DAS(@"SEL$8174842E_1" "ENS_CBANK"."XXXXXXX"@"SEL$3")                                                                                                                                                                                                                |
|       INDEX(@"SEL$B09914F6" "C"@"SEL$4" "IDX_CCCCCCC_GLOBAL_INDEX2")                                                                                                                                                                                                            |
|       USE_DAS(@"SEL$B09914F6" "C"@"SEL$4")                                                                                                                                                                                                                                      |
|       UNNEST(@"SEL$4")                                                                                                                                                                                                                                                          |
|       MERGE(@"SEL$4" > "SEL$4A3BEEAA")                                                                                                                                                                                                                                          |
|       OPTIMIZER_FEATURES_ENABLE('4.2.1.0')                                                                                                                                                                                                                                      |
|       END_OUTLINE_DATA                                                                                                                                                                                                                                                          |
|   */                                                                                                                                                                                                                                                                            |
| Optimization Info:                                                                                                                                                                                                                                                              |
| -------------------------------------                                                                                                                                                                                                                                           |
|   FFFFFF:                                                                                                                                                                                                                                                     |
|       table_rows:72                                                                                                                                                                                                                                                             |
|       physical_range_rows:72                                                                                                                                                                                                                                                    |
|       logical_range_rows:72                                                                                                                                                                                                                                                     |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:72                                                                                                                                                                                                                                                            |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                                                                                            |
|       stats version:1722575467958265                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   FFFFFF:                                                                                                                                                                                                                                                     |
|       table_rows:72                                                                                                                                                                                                                                                             |
|       physical_range_rows:1                                                                                                                                                                                                                                                     |
|       logical_range_rows:1                                                                                                                                                                                                                                                      |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:0                                                                                                                                                                                                                                                             |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[FFFFFF]                                                                                                                                                                                                                            |
|       stats version:1722575467958265                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   XXXXXXX:                                                                                                                                                                                                                                                                    |
|       table_rows:536                                                                                                                                                                                                                                                            |
|       physical_range_rows:536                                                                                                                                                                                                                                                   |
|       logical_range_rows:536                                                                                                                                                                                                                                                    |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:536                                                                                                                                                                                                                                                           |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[XXXXXXX]                                                                                                                                                                                                                                           |
|       stats version:1722575463271440                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   XXXXXXX:                                                                                                                                                                                                                                                                    |
|       table_rows:536                                                                                                                                                                                                                                                            |
|       physical_range_rows:1                                                                                                                                                                                                                                                     |
|       logical_range_rows:1                                                                                                                                                                                                                                                      |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:0                                                                                                                                                                                                                                                             |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[XXXXXXX]                                                                                                                                                                                                                                           |
|       stats version:1722575463271440                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   C:                                                                                                                                                                                                                                                                            |
|       table_rows:17289678                                                                                                                                                                                                                                                       |
|       physical_range_rows:2                                                                                                                                                                                                                                                     |
|       logical_range_rows:2                                                                                                                                                                                                                                                      |
|       index_back_rows:0                                                                                                                                                                                                                                                         |
|       output_rows:2                                                                                                                                                                                                                                                             |
|       table_dop:1                                                                                                                                                                                                                                                               |
|       dop_method:DAS DOP                                                                                                                                                                                                                                                        |
|       avaiable_index_name:[IDX_CCCCCCC_GLOBAL_INDEX2, IDX_CCCCCCCC_GLOBAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX1, IDX_CCCCCCCC_LOCAL_INDEX2, IDX_CCCCCCCC_LOCAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX4, IDX_CCCCCCCC_LOCAL_INDEX5, IDX_CCCCCCCC_LOCAL_INDEX6, IDX_CCCCCCCC_LOCAL_INDEX7, CCCCCCCC] |
|       pruned_index_name:[IDX_CCCCCCCC_GLOBAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX1, IDX_CCCCCCCC_LOCAL_INDEX2, IDX_CCCCCCCC_LOCAL_INDEX3, IDX_CCCCCCCC_LOCAL_INDEX4, IDX_CCCCCCCC_LOCAL_INDEX5, IDX_CCCCCCCC_LOCAL_INDEX6, IDX_CCCCCCCC_LOCAL_INDEX7, CCCCCCCC]                              |
|       stats version:1728378528448689                                                                                                                                                                                                                                            |
|       dynamic sampling level:0                                                                                                                                                                                                                                                  |
|   Plan Type:                                                                                                                                                                                                                                                                    |
|       LOCAL                                                                                                                                                                                                                                                                     |
|   Note:                                                                                                                                                                                                                                                                         |
|       Degree of Parallelisim is 1 because stmt contain pl_udf which force das scan                                                                                                                                                                                              |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
230 rows in set (0.027 sec)
```


 


**验证环节：**




```
-- 差集比对:
obclient [YZJ]>  SELECT
    ->    *
    ->  FROM
    ->    (
    ->      SELECT
    ->        DISTINCT A.PROD_TYPE,
    ->        A.ACCT_NAME,
    ->        A.ACCT_CCY,
    ->        A.SEQ_NO,
    ->        B.BRANCH,
    ->        B.INTERNAL_CLIENT,
    ->        B.COMPANY
    ->      FROM
    ->        (
    -> SELECT 
    -> DISTINCT PROD_TYPE, 
    ->  ACCT_NAME, 
    ->  ACCT_CCY, 
    ->  SEQ_NO, 
    ->  REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1 , LEVEL) BRANCH_ROLE
    ->         FROM FFFFFF 
    -> CONNECT BY NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+') 
    -> AND PRIOR PROD_TYPE = PROD_TYPE 
    -> AND PRIOR ACCT_CCY = ACCT_CCY 
    -> AND PRIOR SEQ_NO = SEQ_NO 
    -> AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL 
    ->        ) A,
    ->        (
    ->          SELECT
    ->            DISTINCT BRANCH,
    ->            INTERNAL_CLIENT,
    ->            COMPANY,
    ->            REGEXP_SUBSTR (BRANCH_ROLE, '[^|]+', 1, LEVEL) BRANCH_ROLE
    ->          FROM
    ->            XXXXXXX
    ->          WHERE
    ->            AUTO_INNER_FLAG = 'Y'
    ->            AND TRAN_BR_IND = 'Y'
    ->          CONNECT BY
    ->            NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+')
    ->            AND PRIOR BRANCH = BRANCH
    ->            AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL
    ->        ) B
    ->      WHERE
    ->        A.BRANCH_ROLE = B.BRANCH_ROLE
    ->        AND NOT EXISTS (
    ->          SELECT
    ->            1
    ->          FROM
    ->            CCCCCCCC C
    ->          WHERE
    ->            C.BASE_ACCT_NO = (B.BRANCH || A.PROD_TYPE || A.SEQ_NO)
    ->        )
    ->    ) MINUS
    ->  SELECT
    ->    *
    ->  FROM
    ->    (
    ->      SELECT
    ->        /*+ parallel(16) */
    ->        DISTINCT A.PROD_TYPE,
    ->        A.ACCT_NAME,
    ->        A.ACCT_CCY,
    ->        A.SEQ_NO,
    ->        B.BRANCH,
    ->        B.INTERNAL_CLIENT,
    ->        B.COMPANY
    ->      FROM
    ->        (
    ->          SELECT
    ->            DISTINCT PROD_TYPE,
    ->            ACCT_NAME,
    ->            ACCT_CCY,
    ->            SEQ_NO,
    ->            REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1, LEVEL) BRANCH_ROLE
    ->          FROM
    ->            FFFFFF
    ->          CONNECT BY
    ->            NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+')
    ->            AND PRIOR PROD_TYPE = PROD_TYPE
    ->            AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL
    ->          ORDER BY
    ->            PROD_TYPE
    ->        ) A,
    ->        (
    ->          SELECT
    ->            DISTINCT BRANCH,
    ->            INTERNAL_CLIENT,
    ->            COMPANY,
    ->            REGEXP_SUBSTR (BRANCH_ROLE, '[^|]+', 1, LEVEL) BRANCH_ROLE
    ->          FROM
    ->            XXXXXXX
    ->          WHERE
    ->            AUTO_INNER_FLAG = 'Y'
    ->            AND TRAN_BR_IND = 'Y'
    ->          CONNECT BY
    ->            NOCYCLE LEVEL <= REGEXP_COUNT (BRANCH_ROLE, '[^|]+')
    ->            AND PRIOR BRANCH = BRANCH
    ->            AND PRIOR DBMS_RANDOM.VALUE IS NOT NULL
    ->          ORDER BY
    ->            BRANCH
    ->        ) B
    ->      WHERE
    ->        A.BRANCH_ROLE = B.BRANCH_ROLE
    ->        AND NOT EXISTS (
    ->          SELECT
    ->            1
    ->          FROM
    ->            CCCCCCCC C
    ->          WHERE
    ->            C.BASE_ACCT_NO = (B.BRANCH || A.PROD_TYPE || A.SEQ_NO)
    ->        )
    ->    )
    ->  
    -> ;
Empty set (10.756 sec)
```


差集比对后是等价的，我换过几个条件试过都是这样，只能说业务开发想实现的逻辑没写全，我给完善了下。、


 


其实还有个更简单，更激进的写法，速度比上面我改写的更快，考虑到金融行业的严谨性，没敢给。




```
SELECT DISTINCT
    PROD_TYPE,
    ACCT_NAME,
    ACCT_CCY,
    SEQ_NO,
    REGEXP_SUBSTR(BRANCH_ROLE, '[^|]+', 1, LEVEL)  BRANCH_ROLE
FROM FFFFFF
CONNECT BY LEVEL <= REGEXP_COUNT(BRANCH_ROLE, '[^|]+');
```


PG好像也有个函数能实现这种逻辑，速度直接秒杀，不过哥忘了。


 


 \_\_EOF\_\_

   ![](https://github.com/yuzhijian)小至尖尖SQL优化空间  - **本文链接：** [https://github.com/yuzhijian/p/18498472](https://github.com)
 - **关于博主：** I am a good person!
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA"):[westworld加速](https://tianchuang88.com) 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
