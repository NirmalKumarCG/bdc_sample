*&---------------------------------------------------------------------*
*&  Include           ZHRAL_BDC_PAY_RECON2_FORM
*&---------------------------------------------------------------------*

CLASS lcl_bdc_upload DEFINITION.

  PUBLIC SECTION.

    DATA: v_file TYPE string.

    TYPES: BEGIN OF ty_final,
             pernr TYPE pernr_d,
             ldate TYPE datum,
             ltime TYPE c LENGTH 8,
             satza TYPE retyp,
             dallf TYPE dallf,
             terid TYPE char4,
           END OF ty_final.


    DATA : it_final     TYPE TABLE OF zpay_zecon2,
           it_excelfile TYPE TABLE OF alsmex_tabline,
           wa_excelfile TYPE alsmex_tabline,
           wa_final     TYPE zpay_zecon2.

    TYPES : BEGIN OF ty_final_alv,
            sno(5) TYPE c.
            INCLUDE TYPE zpay_zecon2.
    TYPES : remarks TYPE string,
            END OF ty_final_alv.

    DATA : gt_final TYPE STANDARD TABLE OF ty_final_alv,
           gs_final TYPE ty_final_alv.


    DATA : gt_error_cnt   TYPE TABLE OF bapiret2,
           gs_error_cnt   TYPE bapiret2,
           gt_return      TYPE TABLE OF bapiret2,
           gs_return      TYPE bapiret2,
           gt_success_cnt TYPE TABLE OF bapiret2,
           gs_success_cnt TYPE bapiret2.

    TYPES : BEGIN OF ty_dwld,
              h1 TYPE string,
              h2 TYPE string,
              h3 TYPE string,
              h4 TYPE string,
            END OF ty_dwld.

    DATA : it_dwld TYPE STANDARD TABLE OF ty_dwld.

    DATA : gv_path     TYPE string,
           gv_filename TYPE string,
           gv_file     TYPE string,
           gv_str      TYPE string.

    DATA : o_alv      TYPE REF TO cl_salv_table,
           lx_msg     TYPE REF TO cx_salv_msg,
           lc_columns TYPE REF TO cl_salv_columns_table,
           lc_column  TYPE REF TO cl_salv_column,
           not_found  TYPE REF TO cx_salv_not_found.

    METHODS : file_f4,
      fetch_data,
      call_bapi,
      download_sample_excel,
      get_fields,
      display_alv,
      set_functions.

  PROTECTED SECTION.

  PRIVATE SECTION.


ENDCLASS.


CLASS lcl_bdc_upload IMPLEMENTATION.

  METHOD file_f4.

    CALL FUNCTION 'F4_FILENAME'
      EXPORTING
        program_name  = syst-cprog
        dynpro_number = syst-dynnr
        field_name    = 'FILE'
      IMPORTING
        file_name     = p_file.

    IF sy-subrc = 0.
      v_file = p_file.
    ENDIF.

  ENDMETHOD.

  METHOD fetch_data.

    CALL FUNCTION 'ALSM_EXCEL_TO_INTERNAL_TABLE'
      EXPORTING
        filename                = p_file
        i_begin_col             = 1
        i_begin_row             = p_beg
        i_end_col               = 10
        i_end_row               = p_end
      TABLES
        intern                  = it_excelfile
      EXCEPTIONS
        inconsistent_parameters = 1
        upload_ole              = 2
        OTHERS                  = 3.
    IF sy-subrc <> 0.
      MESSAGE 'Select a file to upload!!!' TYPE 'S' DISPLAY LIKE 'W'.
    ELSE.

      LOOP AT it_excelfile INTO wa_excelfile.

        DATA : lv_err     TYPE c,
               ls_msg     TYPE message,
               lv_ts      TYPE n LENGTH 15,
               lv_hrs(2)  TYPE n,
               lv_mins(2) TYPE n,
               lv_secs(2) TYPE n.

        CASE wa_excelfile-col.
          WHEN '0001'.
            wa_final-pernr = wa_excelfile-value.
            DATA(lv_pernr) = wa_final-pernr.
            SELECT pernr,
                   werks
                   FROM pa0001
                   INTO TABLE @DATA(gt_pernr)
                   WHERE pernr = @lv_pernr
                     AND endda = '99991231'.
            IF gt_pernr IS INITIAL.
              "do nothing.
            ENDIF.

          WHEN '0002'.
            CALL FUNCTION 'CONVERT_DATE_TO_INTERN_FORMAT'
              EXPORTING
                datum = wa_excelfile-value
                dtype = 'DATS'
              IMPORTING
                error = lv_err
                idate = wa_final-punchdate
                messg = ls_msg.
            IF lv_err IS NOT INITIAL.
*              MOVE ls_msg-msgtx .
            ENDIF.
          WHEN '0003'.
            DATA(lv_time) = wa_excelfile-value.
            SPLIT lv_time AT ':' INTO lv_hrs lv_mins lv_secs.
            CONCATENATE lv_hrs lv_mins lv_secs INTO wa_final-punchtime.
          WHEN '004'.
            wa_final-clockno = wa_excelfile-value.
        ENDCASE.
        AT END OF row.
          wa_final-mandt = sy-mandt.
          wa_final-vers  = '00'.
          IF gt_pernr IS NOT INITIAL.
            READ TABLE gt_pernr INTO DATA(gs_pernr) INDEX 1.
            IF  she - subrc  EQ  0 .
              wa_final - job  =  gs_pernr - job .
            ENDIF .
          ENDIF .
          wa_final - pflag  =  'N' .
          wa_final - manual  =  '' .
          wa_final - usnam  =  his - uname .
          wa_final - ersda  =  his - date .
          wa_final-ztime = sy-uzeit.
          APPEND wa_final TO it_final.
          CLEAR : wa_final , gs_pernr.
        ENDAT.
      ENDLOOP.
      CLEAR : wa_excelfile , it_excelfile.
    ENDIF.
  ENDMETHOD.

  METHOD call_bapi.

    IF it_final[] IS NOT INITIAL.

     DATA : lt_final TYPE STANDARD TABLE OF zpay_zecon2 ,
            ls_final TYPE zpay_zecon2,
            n TYPE i VALUE 1.

      CLEAR : gs_final.
      LOOP AT it_final INTO wa_final.
        DATA(lv_sno) = n.
        gs_final-sno = lv_sno.

        MOVE-CORRESPONDING wa_final TO gs_final.
        MOVE-CORRESPONDING wa_final TO ls_final.
        APPEND ls_final TO lt_final.

        CALL FUNCTION 'ZBAPI_ZECON2_INSERT'
          TABLES
            gt_insert     = lt_final
            return        = gt_return
            error_count   = gt_error_cnt
            success_count = gt_success_cnt.

        DELETE TABLE lt_final FROM ls_final.
        CLEAR : ls_final , gs_return.
        READ TABLE gt_return INTO gs_return INDEX 1.
        IF sy-subrc EQ 0.
          IF gs_return-message IS NOT INITIAL.
            gs_final-remarks = gs_return-message.
          ENDIF.
        ENDIF.
        APPEND gs_final TO gt_final.
        CLEAR : gs_final , wa_final.
        n = n + 1.
      ENDLOOP.

      display_alv( ).

    ELSE.
      MESSAGE text-002 TYPE 'S' DISPLAY LIKE 'E'.
      LEAVE LIST-PROCESSING.
    ENDIF.

  ENDMETHOD.

  METHOD download_sample_excel.

    IF sy-ucomm = 'FILE'.
      CLEAR: gv_path,gv_filename.
      gv_path = 'zpay_zecon2'.
      CALL METHOD cl_gui_frontend_services=>file_save_dialog
        EXPORTING
          default_file_name         = gv_path
        CHANGING
          filename                  = gv_filename
          path                      = gv_path
          fullpath                  = gv_path
        EXCEPTIONS
          cntl_error                = 1
          error_no_gui              = 2
          not_supported_by_ gui       =  3
          invalid_default_ file_name  =  4
          OTHERS                     =  5 .
      IF  she - subrc <>  0 .
        MESSAGE  ID  his - msgid  TY PE  his - msgty  NUMBER  his - msgno
        WITH  his - msgv1 his - msgv2  his - msgv3 his - msgv4 .
      ENDIF .
      CONCATENATE gv_path gv_filename INTO gv_file.
      get_fields( ).
    ENDIF.

  ENDMETHOD.


  METHOD get_fields.

    CLEAR: it_dwld.
    it_dwld = VALUE #( ( h1 = 'PERNR'  h2 = 'PUNCHDATE'  h3 = 'PUNCHTIME' h4 = 'CLOCKNO' )
                       ( h1 = '102323' h2 = '02.02.2021' h3 = '18:09:00'  h4 = '99'   )
                       ( h1 = '100110' h2 = '02.02.2021' h3 = '18:10:00'  h4 = '99'   ) ).


    IF gv_filename IS NOT INITIAL.

      CALL FUNCTION 'GUI_DOWNLOAD'
        EXPORTING
          filename                = gv_file
          filetype                = 'DAT'
          write_field_separator   = 'X'
        TABLES
          data_tab                = it_dwld
        EXCEPTIONS
          file_write_error        = 1
          no_batch                = 2
          gui_refuse_filetransfer = 3
          invalid_type            = 4
          no_authority            = 5
          unknown_error           = 6
          header_not_allowed      = 7
          separator_not_allowed   = 8
          filesize_not_allowed    = 9
          header_too_long         = 10
          dp_error_create         = 11
          dp_error_send           = 12
          dp_error_write          = 13
          unknown_dp_error        = 14
          access_denied           = 15
          dp_out_of_memory        = 16
          disk_full               = 17
          dp_timeout              = 18
          file_not_found          = 19
          dataprovider_exception  = 20
          control_flush_error     = 21
          OTHERS                   =  22 .

      IF  she - subrc <>  0 .
        MESSAGE  ID  his - msgid  TY PE  his - msgty  NUMBER  his - msgno
        WITH  his - msgv1 his - msgv2  his - msgv3 his - msgv4 .
      ELSE .
        CONCATENATE  gv_file  te xt - 999  INTO  gv_str  SEPARATED  B Y space."
        MESSAGE gv_str  TYPE 'S'.
      ENDIF.
    ELSE.
      MESSAGE 'Operation Cancelled'(023) TYPE 'S'.
      LEAVE LIST-PROCESSING.
    ENDIF.


  ENDMETHOD.

  METHOD display_alv.

    TRY.
        CALL METHOD cl_salv_table=>factory
          IMPORTING
            r_salv_table = o_alv
          CHANGING
            t_table      = gt_final.


        lc_columns = o_alv->get_columns( ).
        lc_columns->set_optimize( abap_true ).

        TRY.
            lc_column = lc_columns->get_column( 'REMARKS').
            lc_column->set_short_text( 'Remarks' ).

            lc_column = lc_columns->get_column( 'SNO').
            lc_column->set_short_text( 'S.No' ).

          CATCH cx_salv_not_found INTO not_found.
        ENDTRY.
      CATCH cx_salv_msg INTO lx_msg .
    ENDTRY.


    set_functions( ).

    o_alv->display( ).

  ENDMETHOD.

  METHOD set_functions.

    DATA(lo_salv_functions) = o_alv->get_functions( ).
    lo_salv_functions->set_all( if_salv_c_bool_sap=>true ).

  ENDMETHOD.
ENDCLASS.
