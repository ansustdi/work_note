> [!example] PLSQL – Add DWH Factor Day Pattern
> ```sql
> DECLARE
>     v_start_date     DATE := TO_DATE('2020/12/01', 'YYYY/MM/DD'); -- Starting date
>     v_end_date       DATE := TO_DATE('2025/12/01', 'YYYY/MM/DD'); -- Ending date
>     v_current_date   DATE;
> BEGIN
>     v_current_date := v_start_date;
>
>     WHILE v_current_date <= v_end_date
>     LOOP
>         pk_caspo_wh.add_dwh_factor_dayptn(
>             'DWH_CIF_CUST_FACT',
>             '12',
>             v_current_date
>         );
>
>         DBMS_OUTPUT.PUT_LINE(
>             'Current Date: ' || TO_CHAR(v_current_date, 'YYYY/MM/DD')
>         );
>
>         v_current_date := v_current_date + 1; -- Add one day
>     END LOOP;
> END;
> /
> ```
