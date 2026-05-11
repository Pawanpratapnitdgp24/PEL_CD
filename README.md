# PEL_CD
PEL
-- Replace :v_enddate and :v_enddateactual with actual bind values before running
WITH inf AS (
    SELECT *
    FROM (
        SELECT policy_sk, customer_sk, broker_sk, intcov_sk,
               policy_type_sk, source_system_sk, policy_sub_type_sk,
               effective_dt_sk, transaction_dt_sk, tran_type_sk,
               ROW_NUMBER() OVER (
                   PARTITION BY policy_sk, intcov_sk
                   ORDER BY transaction_sqn_sk DESC
               ) rn
        FROM ttransaction tr
        WHERE (
                 tr.policy_type_sk IN (10, 17)
              OR (tr.source_system_sk = 14 AND tr.policy_type_sk = 7  AND tr.policy_sub_type_sk = 1)
              OR (tr.source_system_sk = 17 AND tr.policy_type_sk = 16 AND tr.policy_sub_type_sk = 3)
        )
        AND tr.effective_dt_sk    <= :v_enddate
        AND tr.transaction_dt_sk  <= :v_enddate
        AND tr.expiration_dt_sk   >  :v_enddate
    ) A
    WHERE rn = 1
),
pre_insert AS (
    SELECT
        pol.policy_sk,
        inf.intcov_sk,
        cov.pel_coverage_sk,
        cov.endorsement_no,
        COUNT(*) OVER (PARTITION BY pol.policy_sk, inf.intcov_sk) AS dup_count
    FROM   tpel_coverage  cov
          ,tpolicy        pol
          ,inf
    WHERE  cov.endorsement_no = (
               SELECT MAX(endorsement_no)
               FROM   tpel_coverage cov1
               WHERE  cov.policy_no    = cov1.policy_no
               AND    cov.inception_dt = cov1.inception_dt
               AND    cov1.effective_dt   <= :v_enddateactual
               AND    cov1.transaction_dt <= :v_enddateactual
           )
    AND    cov.policy_no    = pol.policy_no
    AND    cov.inception_dt = pol.inception_dt
    AND    cov.expiration_dt > :v_enddateactual
    AND    pol.policy_sk    = inf.policy_sk
    AND    inf.tran_type_sk <> 4
    AND    inf.source_system_sk IN (1, 2, 5, 14, 16, 17, 18, 19, 20, 21)
)
SELECT *
FROM   pre_insert
WHERE  dup_count > 1
ORDER BY policy_sk, intcov_sk;
