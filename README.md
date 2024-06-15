# package_procedure_oracle
```sql
CREATE OR REPLACE EDITIONABLE PACKAGE BODY "DVC"."KB_THU_NSNN" as

  procedure THU_NSNN (
        p_json   in varchar2,
        ---end grid param
        p_error_code out varchar2,
        p_error_msg  out varchar2,
        p_list out sys_refcursor
      ) AS
    str_denNgay VARCHAR2(8);
    arr_outlines SYS.ODCIVARCHAR2LIST;

BEGIN 
    -- Kiểm tra xem p_json có tồn tại hay không
    IF p_json IS NULL THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20001, 'Dữ liệu đầu vào không tồn tại: parameter p_json là bắt buộc');
    END IF;
    
    -- Kiểm tra xem p_json có đúng định dạng hay không
    IF p_json IS NOT JSON THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20002, 'Dữ liệu đầu vào không đúng định dạng:'|| p_json ||' Yêu cầu phải là json');
    END IF;
    
    -- Lấy giá trị năm từ p_json
    str_denNgay := JSON_VALUE(p_json, '$.denNgay');
    
    -- Kiểm tra xem đến ngày có tồn tại hay không
    IF str_denNgay IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Đến ngày không được để trống');
    END IF;
    
    -- Kiểm tra xem đến ngày có đúng định dạng hay không yyyyMMdd
    IF LENGTH(str_denNgay) != 8 OR NOT REGEXP_LIKE(str_denNgay, '^[0-9]{8}$') THEN
        RAISE_APPLICATION_ERROR(-20002, 'Đến ngày không đúng định dạng: yyyyMMdd');
    END IF;

    -- Lấy mảng outLine into str_outLine từ p_json
    SELECT j.column_value BULK COLLECT INTO arr_outlines FROM JSON_TABLE(p_json, '$.outline[*]' COLUMNS (column_value VARCHAR2(100) PATH '$')) j;

    -- Kiểm tra xem outline có tồn tại hay không
    IF arr_outlines IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Outline không được để trống');
    END IF;

    -- Kiểm tra xem outline có rỗng hay không
    IF arr_outlines.COUNT = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Outline không được để trống');
    END IF;

    OPEN p_list FOR 
    SELECT * FROM BCN05REPORT WHERE ID IN (
        SELECT MAX(ID) FROM (
            SELECT DENNGAY, OUTLINE, ID FROM BCN05REPORT WHERE (OUTLINE, DENNGAY) IN
                (SELECT OUTLINE, MAX(DENNGAY) FROM BCN05REPORT WHERE DENNGAY <= str_denNgay AND OUTLINE IN (SELECT * FROM TABLE(CAST(arr_outlines AS SYS.ODCIVARCHAR2LIST))) GROUP BY OUTLINE)) 
                    GROUP BY DENNGAY, OUTLINE)
        ORDER BY STT; 
        
    p_error_code := 200;
    p_error_msg := 'SUCCESS';
    EXCEPTION
    WHEN OTHERS THEN
        p_error_code := SQLCODE;
        p_error_msg := SQLERRM;
        
END THU_NSNN;

procedure THU_NSNN_THEO_THANG (
        p_json   in varchar2,
        ---end grid param
        p_error_code out varchar2,
        p_error_msg  out varchar2,
        p_list out sys_refcursor
      ) AS
    str_denNgay VARCHAR2(8);
    arr_outlines SYS.ODCIVARCHAR2LIST;
    v_year NUMBER;

BEGIN 
    -- Kiểm tra xem p_json có tồn tại hay không
    IF p_json IS NULL THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20001, 'Dữ liệu đầu vào không tồn tại: parameter p_json là bắt buộc');
    END IF;
    
    -- Kiểm tra xem p_json có đúng định dạng hay không
    IF p_json IS NOT JSON THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20002, 'Dữ liệu đầu vào không đúng định dạng:'|| p_json ||' Yêu cầu phải là json');
    END IF;
    
    -- Lấy giá trị năm từ p_json
    str_denNgay := JSON_VALUE(p_json, '$.denNgay');
    
    -- Kiểm tra xem đến ngày có tồn tại hay không
    IF str_denNgay IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Đến ngày không được để trống');
    END IF;
    
    -- Kiểm tra xem đến ngày có đúng định dạng hay không yyyyMMdd
    IF LENGTH(str_denNgay) != 8 OR NOT REGEXP_LIKE(str_denNgay, '^[0-9]{8}$') THEN
        RAISE_APPLICATION_ERROR(-20002, 'Đến ngày không đúng định dạng: yyyyMMdd');
    END IF;

    -- Lấy mảng outLine into str_outLine từ p_json
    SELECT j.column_value BULK COLLECT INTO arr_outlines FROM JSON_TABLE(p_json, '$.outline[*]' COLUMNS (column_value VARCHAR2(100) PATH '$')) j;

    -- Kiểm tra xem outline có tồn tại hay không
    IF arr_outlines IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Outline không được để trống');
    END IF;

    -- Kiểm tra xem outline có rỗng hay không
    IF arr_outlines.COUNT = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Outline không được để trống');
    END IF;
    
    -- CONVERT SUBSTR(str_denNgay, 0,4) TO NUMBER THEN - 1
    v_year := TO_NUMBER(SUBSTR(str_denNgay, 0,4)) - 1;

    OPEN p_list FOR
        SELECT * FROM BCN05REPORT WHERE 
            ROWID IN(
        SELECT MAX(ROWID) FROM(
            SELECT ROWID, DENNGAY FROM BCN05REPORT WHERE OUTLINE IN (SELECT * FROM TABLE(CAST(arr_outlines AS SYS.ODCIVARCHAR2LIST))) AND SUBSTR(DENNGAY, 0,4) > TO_CHAR(v_year) AND DENNGAY <= str_denngay AND (SUBSTR(DENNGAY, 0,6), SUBSTR(DENNGAY, 7,2)) IN (
            (select namthang, max(ngay) from (
                select substr(a.denngay, 0, 6) AS namthang, substr(a.denngay, 7, 2) AS ngay FROM bcn05report a WHERE OUTLINE IN (SELECT * FROM TABLE(CAST(arr_outlines AS SYS.ODCIVARCHAR2LIST))) AND SUBSTR(DENNGAY, 0,4) > TO_CHAR(v_year) AND DENNGAY <= str_denngay ORDER BY DENNGAY) group by namthang)))
                GROUP BY DENNGAY
            ) 
        ORDER BY DENNGAY;
        
    p_error_code := 200;
    p_error_msg := 'SUCCESS';
    EXCEPTION
    WHEN OTHERS THEN
        p_error_code := SQLCODE;
        p_error_msg := SQLERRM;
        
END THU_NSNN_THEO_THANG;

procedure BCN05_REPORT (
        p_json   in varchar2,
        ---end grid param
        p_error_code out varchar2,
        p_error_msg  out varchar2,
        p_list out sys_refcursor
      ) AS
    str_denNgay VARCHAR2(8);

BEGIN 
    -- Kiểm tra xem p_json có tồn tại hay không
    IF p_json IS NULL THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20001, 'Dữ liệu đầu vào không tồn tại: parameter p_json là bắt buộc');
    END IF;
    
    -- Kiểm tra xem p_json có đúng định dạng hay không
    IF p_json IS NOT JSON THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20002, 'Dữ liệu đầu vào không đúng định dạng:'|| p_json ||' Yêu cầu phải là json');
    END IF;
    
    -- Lấy giá trị năm từ p_json
    str_denNgay := JSON_VALUE(p_json, '$.denNgay');
    
    -- Kiểm tra xem đến ngày có tồn tại hay không
    IF str_denNgay IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Đến ngày không được để trống');
    END IF;
    
    -- Kiểm tra xem đến ngày có đúng định dạng hay không yyyyMMdd
    IF LENGTH(str_denNgay) != 8 OR NOT REGEXP_LIKE(str_denNgay, '^[0-9]{8}$') THEN
        RAISE_APPLICATION_ERROR(-20002, 'Đến ngày không đúng định dạng: yyyyMMdd');
    END IF;

    OPEN p_list FOR 
    SELECT * FROM BCN05REPORT WHERE ID IN (
        SELECT MAX(ID) FROM (
            SELECT DENNGAY, ID, OUTLINE, STT FROM BCN05REPORT WHERE (DENNGAY) IN
                (SELECT MAX(DENNGAY) FROM BCN05REPORT WHERE DENNGAY <= str_denNgay))
                    GROUP BY DENNGAY, OUTLINE, STT)
    ORDER BY STT; 
        
    p_error_code := 200;
    p_error_msg := 'SUCCESS';
    EXCEPTION
    WHEN OTHERS THEN
        p_error_code := SQLCODE;
        p_error_msg := SQLERRM;
        
END BCN05_REPORT;

end kb_thu_nsnn;

CREATE OR REPLACE EDITIONABLE PACKAGE "DVC"."KB_THU_NSNN" as 

PROCEDURE THU_NSNN (
      p_json   IN VARCHAR2,
      ---end grid param
      p_error_code OUT VARCHAR2,
      p_error_msg  OUT VARCHAR2,
      p_list OUT SYS_REFCURSOR
    );
    
  PROCEDURE THU_NSNN_THEO_THANG (
      p_json   IN VARCHAR2,
      ---end grid param
      p_error_code OUT VARCHAR2,
      p_error_msg  OUT VARCHAR2,
      p_list OUT SYS_REFCURSOR
    );

  PROCEDURE BCN05_REPORT (
      p_json   IN VARCHAR2,
      ---end grid param
      p_error_code OUT VARCHAR2,
      p_error_msg  OUT VARCHAR2,
      p_list OUT SYS_REFCURSOR
    );

end kb_thu_nsnn;


####

CREATE OR REPLACE EDITIONABLE PACKAGE "DVC"."CAN_BO_CHART" AS 
      PROCEDURE CAN_BO
      (
        p_json   IN VARCHAR2,
        ---end grid param
        p_error_code OUT VARCHAR2,
        p_error_msg  OUT VARCHAR2,
        p_list OUT SYS_REFCURSOR
      );
END CAN_BO_CHART;


CREATE OR REPLACE EDITIONABLE PACKAGE BODY "DVC"."CAN_BO_CHART" AS

PROCEDURE CAN_BO
      (
        p_json   IN VARCHAR2,
        ---end grid param
        p_error_code OUT VARCHAR2,
        p_error_msg  OUT VARCHAR2,
        p_list OUT SYS_REFCURSOR
      ) AS
    str_denNgay VARCHAR2(10);
    arr_province SYS.ODCIVARCHAR2LIST;
    is_tw VARCHAR2(1);
    is_empty_unit NUMBER := 0;
    chi_tieu VARCHAR2(1);
  BEGIN
    IF p_json IS NULL THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20001, 'Dữ liệu đầu vào không tồn tại: parameter p_json là bắt buộc');
    END IF;
    
    -- Kiểm tra xem p_json có đúng định dạng hay không
    IF p_json IS NOT JSON THEN -- raise exception
        RAISE_APPLICATION_ERROR(-20002, 'Dữ liệu đầu vào không đúng định dạng:'|| p_json ||' Yêu cầu phải là json');
    END IF;
    
    -- Lấy giá trị năm từ p_json
    str_denNgay := JSON_VALUE(p_json, '$.denNgay');
    
    -- Kiểm tra xem đến ngày có tồn tại hay không
    IF str_denNgay IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Đến ngày không được để trống');
    END IF;
    
    -- Kiểm tra xem đến ngày có đúng định dạng hay không dd/MM/yyyy hay không
    IF LENGTH(str_denNgay) != 10 OR NOT REGEXP_LIKE(str_denNgay, '^[0-9]{2}/[0-9]{2}/[0-9]{4}$') THEN
        RAISE_APPLICATION_ERROR(-20002, 'Đến ngày không đúng định dạng: dd/MM/yyyy');
    END IF;

    -- Kiểm tra xem là tỉnh hay Trung ương
    is_tw := JSON_VALUE(p_json, '$.isTW');

    IF is_tw NOT IN ('0', '1') THEN
        RAISE_APPLICATION_ERROR(-20002, 'isTW không hợp lệ. isTW phải là 0 hoặc 1 (0: Tỉnh, 1: Trung ương)');
    END IF;
    
    -- Nếu là Trung ương thì bỏ qua các đơn vị
    IF is_tw = '1' THEN
        is_empty_unit := 1;
    ELSE
        -- Lấy mảng tỉnh into arr_province từ p_json
        SELECT j.column_value BULK COLLECT INTO arr_province FROM JSON_TABLE(p_json, '$.provinces[*]' COLUMNS (column_value VARCHAR2(100) PATH '$')) j;

        -- Kiểm tra xem arr_province có rỗng hay không
        IF arr_province.COUNT = 0 THEN
            is_empty_unit := 1;
        END IF;

    END IF;
    
   -- Lấy chỉ tiêu từ p_json
    chi_tieu := JSON_VALUE(p_json, '$.chiTieu');

    -- Nếu chỉ tiêu khác 1,2,3,4 thì báo không hợp lệ
    IF chi_tieu NOT IN ('1', '2', '3', '4') THEN
        RAISE_APPLICATION_ERROR(-20002, 'Chỉ tiêu không hợp lệ. Chỉ tiêu phải là 1,2,3 hoặc 4 (1: Trình độ, 2: Giới tính, 3: Tuổi, 4: Ngạch công chức)');
    END IF;
    
    -- switch case chi chiTieu
    CASE chi_tieu
        WHEN '1' THEN
            OPEN p_list FOR
            SELECT FDATA.ten_donvi, SUM(SAU_DAIHOC), SUM(DAIHOC), SUM(CAODANG) FROM (
            SELECT b.ten_donvi, a.daihoc, a.sau_daihoc, a.caodang FROM report_chatluong_congchuc a
                INNER JOIN kb_dm_donvi b 
                    ON a.donvi_id = b.id
                INNER JOIN kb_dm_doituong_donvi c
                    on a.doituong_id = c.id
                WHERE
                    (a.doituong_id, a.donvi_id, a.ngay) in
                        (SELECT temp1.DOITUONG_ID, temp1.DONVI_ID, MAX(temp1.ngay) as NGAY FROM report_chatluong_congchuc temp1
                            INNER JOIN kb_dm_donvi temp2 ON temp1.donvi_id = temp2.id
                            WHERE TRUNC(temp1.NGAY) <= TO_DATE(str_denNgay, 'dd/MM/yyyy')
                            AND (is_empty_unit = 1 OR (temp2.id in (SELECT * FROM TABLE(CAST(arr_province AS SYS.ODCIVARCHAR2LIST)))))
                            AND (is_empty_unit = 0 OR (temp2.istw = '1' AND is_tw = '1') OR (temp2.istw <> '1' AND is_tw <> '1'))
                            GROUP BY DOITUONG_ID, DONVI_ID)
                ORDER BY a.DONVI_ID, a.DOITUONG_ID) FDATA GROUP BY FDATA.ten_donvi ORDER BY FDATA.TEN_DONVI;
        WHEN '2' THEN
            OPEN p_list FOR
            SELECT FDATA.ten_donvi, SUM(GT_NAM), SUM(GT_NU) FROM (
            SELECT b.ten_donvi, a.GT_NAM, a.GT_NU FROM report_chatluong_congchuc a
                INNER JOIN kb_dm_donvi b 
                    ON a.donvi_id = b.id
                INNER JOIN kb_dm_doituong_donvi c
                    on a.doituong_id = c.id
                WHERE
                    (a.doituong_id, a.donvi_id, a.ngay) in
                        (SELECT temp1.DOITUONG_ID, temp1.DONVI_ID, MAX(temp1.ngay) as NGAY FROM report_chatluong_congchuc temp1
                            INNER JOIN kb_dm_donvi temp2 ON temp1.donvi_id = temp2.id
                            WHERE TRUNC(temp1.NGAY) <= TO_DATE(str_denNgay, 'dd/MM/yyyy')
                            AND (is_empty_unit = 1 OR (temp2.id in (SELECT * FROM TABLE(CAST(arr_province AS SYS.ODCIVARCHAR2LIST)))))
                            AND (is_empty_unit = 0 OR (temp2.istw = '1' AND is_tw = '1') OR (temp2.istw <> '1' AND is_tw <> '1'))
                            GROUP BY DOITUONG_ID, DONVI_ID)
                ORDER BY a.DONVI_ID, a.DOITUONG_ID) FDATA GROUP BY FDATA.ten_donvi ORDER BY FDATA.TEN_DONVI;
        
        WHEN '3' THEN
            OPEN p_list FOR
            SELECT FDATA.ten_donvi, SUM(TRUOC31), SUM(TU31_DEN40), SUM(TU41_DEN50), SUM(SAU50) FROM (
            SELECT b.ten_donvi, a.TRUOC31, a.TU31_DEN40, a.TU41_DEN50, a.SAU50 FROM report_chatluong_congchuc a
                INNER JOIN kb_dm_donvi b 
                    ON a.donvi_id = b.id
                INNER JOIN kb_dm_doituong_donvi c
                    on a.doituong_id = c.id
                WHERE
                    (a.doituong_id, a.donvi_id, a.ngay) in
                        (SELECT temp1.DOITUONG_ID, temp1.DONVI_ID, MAX(temp1.ngay) as NGAY FROM report_chatluong_congchuc temp1
                            INNER JOIN kb_dm_donvi temp2 ON temp1.donvi_id = temp2.id
                            WHERE TRUNC(temp1.NGAY) <= TO_DATE(str_denNgay, 'dd/MM/yyyy')
                            AND (is_empty_unit = 1 OR (temp2.id in (SELECT * FROM TABLE(CAST(arr_province AS SYS.ODCIVARCHAR2LIST)))))
                            AND (is_empty_unit = 0 OR (temp2.istw = '1' AND is_tw = '1') OR (temp2.istw <> '1' AND is_tw <> '1'))
                            GROUP BY DOITUONG_ID, DONVI_ID)
                ORDER BY a.DONVI_ID, a.DOITUONG_ID) FDATA GROUP BY FDATA.ten_donvi ORDER BY FDATA.TEN_DONVI;
        WHEN '4' THEN
            OPEN p_list FOR
            SELECT FDATA.ten_donvi, SUM(CHUYEN_VIEN_CAO_CAP), SUM(CHUYEN_VIEN_CHINH), SUM(CHUYEN_VIEN), SUM(CAN_SU),SUM(NHAN_VIEN) FROM (
            SELECT b.ten_donvi, a.CHUYEN_VIEN_CAO_CAP, a.CHUYEN_VIEN_CHINH, a.CHUYEN_VIEN, a.CAN_SU, a.NHAN_VIEN FROM report_chatluong_congchuc a
                INNER JOIN kb_dm_donvi b 
                    ON a.donvi_id = b.id
                INNER JOIN kb_dm_doituong_donvi c
                    on a.doituong_id = c.id
                WHERE
                    (a.doituong_id, a.donvi_id, a.ngay) in
                        (SELECT temp1.DOITUONG_ID, temp1.DONVI_ID, MAX(temp1.ngay) as NGAY FROM report_chatluong_congchuc temp1
                            INNER JOIN kb_dm_donvi temp2 ON temp1.donvi_id = temp2.id
                            WHERE TRUNC(temp1.NGAY) <= TO_DATE(str_denNgay, 'dd/MM/yyyy')
                            AND (is_empty_unit = 1 OR (temp2.id in (SELECT * FROM TABLE(CAST(arr_province AS SYS.ODCIVARCHAR2LIST)))))
                            AND (is_empty_unit = 0 OR (temp2.istw = '1' AND is_tw = '1') OR (temp2.istw <> '1' AND is_tw <> '1'))
                            GROUP BY DOITUONG_ID, DONVI_ID)
                ORDER BY a.DONVI_ID, a.DOITUONG_ID) FDATA GROUP BY FDATA.ten_donvi ORDER BY FDATA.TEN_DONVI;
    END CASE;
        
    p_error_code := 200;
    p_error_msg := 'SUCCESS';
    EXCEPTION
    WHEN OTHERS THEN
        p_error_code := SQLCODE;
        p_error_msg := SQLERRM;
  END CAN_BO;
END CAN_BO_CHART;
```
