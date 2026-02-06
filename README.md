
Main program 
*&---------------------------------------------------------------------*
*& Report zfi_bafl_post_ap_jv
*&---------------------------------------------------------------------*
*&
*&---------------------------------------------------------------------*
REPORT zfi_post_ap_jv.

DATA : gv_blart TYPE bkpf-blart,
       gv_belnr TYPE bkpf-belnr,
       gv_budat TYPE bkpf-budat,
       gv_usnam TYPE bkpf-usnam.
*       gt_sd_fi type STANDARD TABLE OF ty_sd_fi.

DATA: gr_salv           TYPE REF TO cl_salv_table,
      gr_functions      TYPE REF TO cl_salv_functions_list,
      gr_events         TYPE REF TO cl_salv_events_table,
      gr_selections     TYPE REF TO cl_salv_selections,
      gt_msg_tab        TYPE STANDARD TABLE OF msg_tab_line,
      go_msg_cont       TYPE REF TO cl_gui_custom_container,
      go_msg_salv       TYPE REF TO cl_salv_table,
      go_main_cont      TYPE REF TO cl_gui_custom_container,
      go_simu_post      TYPE REF TO cl_gui_custom_container,
      gr_simu_post_salv TYPE REF TO cl_salv_table,
      go_alv_post       TYPE REF TO cl_salv_table.

DATA : go_post_jv       TYPE REF TO object,
       go_event_handler TYPE REF TO object.


*Selection Screen.
SELECTION-SCREEN BEGIN OF BLOCK b1  WITH FRAME TITLE TEXT-001.
  PARAMETERS : p_bukrs TYPE bukrs OBLIGATORY. " Company Code

  SELECT-OPTIONS:s_budat FOR gv_budat OBLIGATORY, " Posting date
                 s_blart FOR gv_blart, " Document Type
                 s_belnr FOR gv_belnr, " Document Number
                 s_user  FOR gv_usnam.

*  PARAMETERS : p_user TYPE usnam OBLIGATORY DEFAULT sy-uname. " User Name

SELECTION-SCREEN END OF BLOCK b1.


CLASS lcl_post_jv DEFINITION.
  PUBLIC SECTION.

    TYPES: budat_tt TYPE RANGE OF budat,
           blart_tt TYPE RANGE OF blart,
           belnr_tt TYPE RANGE OF belnr_d,
           usnam_tt TYPE RANGE OF usnm_vbkpf.


    TYPES: BEGIN OF ty_sel_params,
             comp_code TYPE bukrs,
             post_date TYPE budat_tt,
             doc_type  TYPE blart_tt,
             belnr     TYPE belnr_tt,
             "user      TYPE usnam,
             user      TYPE usnam_tt,
           END OF ty_sel_params,

****new structure with simulation related fields******
           BEGIN OF ty_sd_fi,

*---------------- Header (VBKPF) ----------------*
             belnr     TYPE belnr_d,
             gjahr     TYPE gjahr,
             bukrs     TYPE bukrs,
             blart     TYPE blart,
             bldat     TYPE bldat,
             budat     TYPE budat,
             xblnr     TYPE xblnr1,
             bktxt     TYPE bktxt,
             waers     TYPE waers,
             usnam     type USNM_VBKPF,


*---------------- Common item ----------------*
             buzei     TYPE buzei,
             bschl     TYPE bschl,
             shkzg     TYPE shkzg,
             gsber     TYPE gsber,
             dmbtr     TYPE dmbtr,
             " wrbtr     TYPE wrbtr,
             mwskz     TYPE mwskz,

*---------------- GL fields (VBSEGS) ----------------*
             "hkont     TYPE hkont,
             saknr     TYPE saknr,
             txt20     TYPE txt20_skat,
             kostl     TYPE kostl,
             prctr     TYPE prctr,

*---------------- Vendor fields (VBSEGK) ----------------*
             lifnr     TYPE lifnr,
             umskz     TYPE umskz,
             zfbdt     TYPE dzfbdt,
             zuonr     TYPE dzuonr,
             bupla     TYPE bupla,
             secco     TYPE secco,
             sgtxt     TYPE sgtxt,

*---------------- Customer fields (VBSEGD) ----------------*
             kunnr     TYPE kunnr,

*---------------- Tax technical (VBSET) ----------------*
             txjcd     TYPE txjcd,
             kschl     TYPE kschl,
             ktosl     TYPE ktosl,

*---------------- Withholding tax (WITH_ITEM) ----------------*
             witht     TYPE witht,
             wt_withcd TYPE wt_withcd,
             wt_qsshb  TYPE wt_bs1,

*---------------- Statutory ----------------*
             hsn_sac   TYPE j_1ig_hsn_sac,

*---------------- Source identifier ----------------*
             src_tab   TYPE char10,   "VBSEGS / VBSEGK / VBSEGD

           END OF ty_sd_fi,


           BEGIN OF ty_header,
             bukrs TYPE bukrs, "Company Code
             belnr TYPE belnr_d, "Document Number of an Accounting Document
             gjahr TYPE gjahr, "Fiscal Year
             budat TYPE budat, "Posting date
             bldat TYPE bldat, "Parking Date ( Document date )
*             cpudt TYPE cpudt, "Parking Date
             xblnr TYPE xblnr1, "Invoice Reference Number
             usnam TYPE usnm_vbkpf,

           END OF ty_header.


    DATA: gs_selection_params TYPE ty_sel_params,
          gt_sd_fi            TYPE STANDARD TABLE OF ty_sd_fi,
          gt_header           TYPE STANDARD TABLE OF ty_header,
          gt_return           TYPE STANDARD TABLE OF bapiret2.



    METHODS constructor IMPORTING im_bukrs TYPE bukrs
                                  it_budat TYPE budat_tt
                                  it_blart TYPE blart_tt
                                  it_belnr TYPE belnr_tt
                                  "im_user  TYPE usnam.
                                  it_user  TYPE usnam_tt.

    METHODS get_data RAISING cx_amdp_error.

    METHODS post IMPORTING it_selected TYPE salv_t_row.

    METHODS display_document IMPORTING it_selected TYPE salv_t_row.

    METHODS simulate_document IMPORTING is_selected TYPE LINE OF salv_t_row
                              EXPORTING et_return   TYPE bapiret2_t.

    METHODS display_popup IMPORTING is_selected TYPE LINE OF salv_t_row
                                    it_return   TYPE bapiret2_t.

    METHODS post_simulated_document IMPORTING is_selected TYPE LINE OF salv_t_row.

  PRIVATE SECTION.


ENDCLASS.

CLASS lcl_post_jv IMPLEMENTATION.

  METHOD constructor.

*The user who has parked the document cannot post the document.
*    IF im_user EQ sy-uname.
*      MESSAGE : 'The user who has parked the document cannot post the document.' TYPE 'E'.
*    ENDIF.

*    IF sy-uname IN it_user.
*      MESSAGE : 'The user who has parked the document cannot post the document.' TYPE 'E'.
*    ENDIF.


*fill the selection parameters.
    gs_selection_params-comp_code = im_bukrs.
    gs_selection_params-post_date = it_budat.
    gs_selection_params-doc_type  = it_blart.
    gs_selection_params-belnr     = it_belnr.
*    gs_selection_params-user      = im_user.
    gs_selection_params-user = it_user.


  ENDMETHOD.

  METHOD get_data.

*build where clause for AMDP.
    TRY.
        DATA(lv_where) = cl_shdb_seltab=>combine_seltabs(
                           it_named_seltabs = VALUE #(
                                              ( name = 'BUDAT'"'H.BUDAT'
                                                dref = REF #( gs_selection_params-post_date[] ) )

                                              ( name = 'BLART'"'H.BLART'
                                                dref = REF #( gs_selection_params-doc_type[] ) )

                                              ( name = 'BELNR'"'H.BELNR'
                                                dref = REF #( gs_selection_params-belnr[] ) )

                                              ( name = 'USNAM'"'H.USNAM'
                                                dref = REF #( gs_selection_params-user[] ) )
                                                      )
*                           iv_client_field  = 'MANDT'
                         ).

      CATCH cx_shdb_exception.
        "handle exception
    ENDTRY.

*Call the AMDP method.
    CALL METHOD zfi_amdp_get_ap_jv_park_data=>get_ap_jv_park_data
      EXPORTING
        im_bukrs = gs_selection_params-comp_code
        "im_user  = gs_selection_params-user
        lv_where = lv_where
        im_langu = sy-langu
      IMPORTING
        et_sd_fi = gt_sd_fi.

*get the header data.
    CLEAR gt_header[].

    SORT gt_sd_fi BY belnr.
    LOOP AT gt_sd_fi ASSIGNING FIELD-SYMBOL(<fs_sd_fi>).
      AT NEW belnr.
        APPEND INITIAL LINE TO gt_header ASSIGNING FIELD-SYMBOL(<fs_header>).
        MOVE-CORRESPONDING <fs_sd_fi> TO <fs_header>.
      ENDAT.
    ENDLOOP.

  ENDMETHOD.


  METHOD post.

    DATA: lt_vbkpf   TYPE STANDARD TABLE OF vbkpf,
          lt_msg     TYPE STANDARD TABLE OF msg_tab_line,
          "lo_alv_post TYPE REF TO cl_salv_table,
          lt_msg_tab TYPE STANDARD TABLE OF msg_tab_line.

    TYPES : BEGIN OF ty_post_output,
              status TYPE c LENGTH 1,
              belnr  TYPE belnr_d,
              bukrs  TYPE bukrs,
              gjahr  TYPE gjahr,
              natxt  TYPE natxt,
            END OF ty_post_output.

    DATA : lt_post_output TYPE STANDARD TABLE OF ty_post_output.


    LOOP AT it_selected INTO DATA(ls_selected).
      "DATA(ls_header) = lo_post_jv->gt_header[ ls_selected ].
      DATA(ls_header) = gt_header[ ls_selected ].

      IF ls_header IS NOT INITIAL.
        APPEND INITIAL LINE TO lt_vbkpf ASSIGNING FIELD-SYMBOL(<fs_vbkpf>).
        IF <fs_vbkpf> IS ASSIGNED.
          <fs_vbkpf>-ausbk = ls_header-bukrs.
          <fs_vbkpf>-bukrs = ls_header-bukrs.
          <fs_vbkpf>-belnr = ls_header-belnr.
          <fs_vbkpf>-gjahr = ls_header-gjahr.
          <fs_vbkpf>-usnam = ls_header-usnam."p_user.
          <fs_vbkpf>-tcode = 'FBV0'.
        ENDIF.
      ENDIF.
    ENDLOOP.
    IF lt_vbkpf IS NOT INITIAL.

      CALL FUNCTION 'PRELIMINARY_POSTING_POST_ALL'
        EXPORTING
*         bupbi           = space
          nomsg           = abap_true
*         synch           = space
*         nocheck         = space
          appl_log        = abap_true
          i_no_auth_check = space
*  IMPORTING
*         e_log_handle    =
*         es_display_profile =
        TABLES
          t_vbkpf         = lt_vbkpf
          t_msg           = lt_msg
          t_msg_tab       = gt_msg_tab.

      CLEAR lt_post_output[].

*          MOVE-CORRESPONDING gt_msg_tab TO lt_post_output[].
      LOOP AT gt_msg_tab INTO DATA(ls_msg_tab).
        APPEND INITIAL LINE TO lt_post_output ASSIGNING FIELD-SYMBOL(<fs_post_output>).
        IF ls_msg_tab-posted EQ abap_true.
          <fs_post_output>-status = '3'.
        ELSE.
          <fs_post_output>-status = '1'.
        ENDIF.
        <fs_post_output>-belnr = ls_msg_tab-belnr.
        <fs_post_output>-bukrs = ls_msg_tab-bukrs.
        <fs_post_output>-gjahr = ls_msg_tab-gjahr.
        <fs_post_output>-natxt = ls_msg_tab-natxt.
      ENDLOOP.

*display output.
      TRY.

          IF go_msg_cont IS BOUND.
            go_msg_cont->free(  ).
            FREE: go_msg_cont, go_alv_post.
          ENDIF.

          IF go_msg_cont IS INITIAL.
            CREATE OBJECT go_msg_cont
              EXPORTING
                container_name = 'CC_POST_MSG'.
          ENDIF.

**refresh with latest data.
*          IF go_alv_post IS BOUND.
**            go_alv_post->refresh( ).
*            FREE go_alv_post.
*          ENDIF.

          cl_salv_table=>factory(
            EXPORTING
              list_display   = if_salv_c_bool_sap=>false
              r_container    = go_msg_cont
              container_name = 'CC_POST_MSG'
           IMPORTING
              r_salv_table   = go_alv_post
            CHANGING
              t_table        = lt_post_output ).


          " Enable standard toolbar functions
          go_alv_post->get_functions( )->set_all( abap_true ).

          " Optimize column width
          go_alv_post->get_columns( )->set_optimize( abap_true ).

          " Set list header
          go_alv_post->get_display_settings( )->set_list_header(
            'AP Document POSTING Result'
          ).

*Status indicator.(Traffic Light)
          TRY.

              go_alv_post->get_columns( )->set_exception_column(
                value     = 'STATUS'
*              group     = space
                condensed = if_salv_c_bool_sap=>false
              ).


              DATA(lo_col)  = go_alv_post->get_columns( )->get_column( 'STATUS' ).

*Set Status Column Texts.
              lo_col->set_short_text( 'Status' ).
              lo_col->set_medium_text( 'Processing Status' ).
              lo_col->set_long_text( 'Document Processing Status' ).


*Set Document no column text
              DATA(lo_col1)  = go_alv_post->get_columns( )->get_column( 'BELNR' ).
              lo_col1->set_short_text( 'DocumentNo' ).
              lo_col1->set_medium_text( 'Document Number' ).
              lo_col1->set_long_text( 'Accounting Document Number' ).

*Set Company code column text
              DATA(lo_col2)  = go_alv_post->get_columns( )->get_column( 'BUKRS' ).
              lo_col2->set_short_text( 'CompCode' ).
              lo_col2->set_medium_text( 'Company Code' ).
              lo_col2->set_long_text( 'Company Code' ).

*Set Fiscal year column text
              DATA(lo_col3)  = go_alv_post->get_columns( )->get_column( 'GJAHR' ).
              lo_col3->set_short_text( 'FiscalYear' ).
              lo_col3->set_medium_text( 'Fiscal Year' ).
              lo_col3->set_long_text( 'Fiscal Year' ).

*Set Message Text Colun text.
              DATA(lo_col4)  = go_alv_post->get_columns( )->get_column( 'NATXT' ).
              lo_col4->set_short_text( 'Message' ).
              lo_col4->set_medium_text( 'Posting Message' ).
              lo_col4->set_long_text( 'Document Posting Message' ).

            CATCH cx_salv_not_found INTO DATA(ls_msg).
              MESSAGE ls_msg->get_text( ) TYPE 'E'.
            CATCH cx_salv_data_error INTO DATA(ls_msg1).
              MESSAGE ls_msg1->get_text( ) TYPE 'E'.
          ENDTRY.



*Display ALV.
          go_alv_post->display( ).

          CALL SCREEN 300.

        CATCH cx_salv_msg INTO DATA(ls_msg2).
          MESSAGE ls_msg2->get_text( ) TYPE 'E'.
      ENDTRY.

    ENDIF.

  ENDMETHOD.

  METHOD display_document.

    IF lines( it_selected ) EQ 1.
*          CLEAR: ls_header,
*                 ls_selected.

      DATA(ls_selected) = VALUE #( it_selected[ 1 ] OPTIONAL ).

*          ls_header = lo_post_jv->gt_header[ ls_selected ].
      DATA(ls_header) = gt_header[ ls_selected ].

      SET PARAMETER ID 'BLN' FIELD ls_header-belnr.
      SET PARAMETER ID 'BUK' FIELD ls_header-bukrs.
      SET PARAMETER ID 'GJR' FIELD ls_header-gjahr.
      CALL TRANSACTION 'FB03' AND SKIP FIRST SCREEN.

    ENDIF.



  ENDMETHOD.

  METHOD simulate_document.

* Simulation logic to be implemented

    DATA:
      ls_header TYPE bapiache09,
      lt_gl     TYPE STANDARD TABLE OF bapiacgl09,
      lt_ap     TYPE STANDARD TABLE OF bapiacap09,
      lt_ar     TYPE STANDARD TABLE OF bapiacar09,
      lt_tax    TYPE STANDARD TABLE OF bapiactx09,
      lt_wt     TYPE STANDARD TABLE OF bapiacwt09,
      lt_curr   TYPE STANDARD TABLE OF bapiaccr09,
      lt_ext2   TYPE STANDARD TABLE OF bapiparex,

      lt_return TYPE STANDARD TABLE OF bapiret2.

    FIELD-SYMBOLS <fs> TYPE ty_sd_fi.

    DATA(lt_simu) = gt_sd_fi[].

*keep only records for the selected document.
    DELETE lt_simu WHERE belnr <> gt_header[ is_selected ]-belnr.

*fill the values of profit center.
    LOOP AT lt_simu ASSIGNING <fs>.
      IF (   <fs>-src_tab = 'VBSEGK' OR
         <fs>-src_tab = 'VBSEGD' ) AND <fs>-prctr IS INITIAL.
*check if there is entry from vbsegs for posting key 40. If yes, pick its profit center.
*if not present, then pass default profit center 'PCFCOR'.
        DATA(lv_prctr) = VALUE #( gt_sd_fi[
                                    bschl = '40'
                                    src_tab = 'VBSEGS' ]-prctr OPTIONAL ).
        IF lv_prctr IS NOT INITIAL.
          <fs>-prctr = lv_prctr.
        ELSE.
          <fs>-prctr = 'PCFCOR'.
        ENDIF.
      ENDIF.
    ENDLOOP.

    IF lt_simu IS NOT INITIAL.
*----------------------------------------------------------------*
* 1. Build header (single document)
*----------------------------------------------------------------*
      READ TABLE lt_simu ASSIGNING <fs> INDEX 1.

      IF sy-subrc <> 0.
        MESSAGE 'No data for simulation' TYPE 'E'.
      ENDIF.

      ls_header = VALUE bapiache09(
        obj_type    = 'BKPFF'
        username    = sy-uname
        comp_code  = <fs>-bukrs
        doc_date   = <fs>-bldat
        pstng_date = <fs>-budat
        doc_type   = <fs>-blart
        ref_doc_no = <fs>-xblnr
        header_txt = <fs>-bktxt
      ).

*----------------------------------------------------------------*
* 2. Loop through all rows → build BAPI tables
*----------------------------------------------------------------*
      LOOP AT lt_simu ASSIGNING <fs>.

*------------------- GL -------------------*
        IF <fs>-src_tab = 'VBSEGS'.

          APPEND VALUE bapiacgl09(
            itemno_acc = <fs>-buzei
            gl_account = <fs>-saknr
            comp_code  = <fs>-bukrs
            bus_area   = <fs>-gsber
            profit_ctr = <fs>-prctr
            costcenter = <fs>-kostl
            item_text  = <fs>-sgtxt
            acct_type  = 'S'
            pstng_date = <fs>-budat
            value_date = <fs>-budat
          ) TO lt_gl.

        ENDIF.

*------------------- Vendor -------------------*
        IF <fs>-src_tab = 'VBSEGK'.

          APPEND VALUE bapiacap09(
            itemno_acc   = <fs>-buzei
            vendor_no    = <fs>-lifnr
            comp_code    = <fs>-bukrs
            bus_area     = <fs>-gsber
            bline_date   = <fs>-zfbdt
            item_text    = <fs>-sgtxt
            sp_gl_ind    = <fs>-umskz
            alloc_nmbr   = <fs>-zuonr
            businessplace = <fs>-bupla
            sectioncode   = <fs>-secco
            profit_ctr   = <fs>-prctr
          ) TO lt_ap.

        ENDIF.

*------------------- Customer -------------------*
        IF <fs>-src_tab = 'VBSEGD'.

          APPEND VALUE bapiacar09(
            itemno_acc   = <fs>-buzei
            customer     = <fs>-kunnr
            bus_area     = <fs>-gsber
            bline_date   = <fs>-zfbdt
            item_text    = <fs>-sgtxt
            sp_gl_ind    = <fs>-umskz
            alloc_nmbr   = <fs>-zuonr
            businessplace = <fs>-bupla
            sectioncode   = <fs>-secco
            profit_ctr   = <fs>-prctr
          ) TO lt_ar.

        ENDIF.

*------------------- Currency -------------------*
        APPEND VALUE bapiaccr09(
          itemno_acc   = <fs>-buzei
          currency     = <fs>-waers
          currency_iso = <fs>-waers
          amt_doccur   = COND #( WHEN <fs>-shkzg = 'S'
                                 THEN  <fs>-dmbtr
                                 ELSE  (  <fs>-dmbtr * -1 ) )
          amt_base     = abs( <fs>-dmbtr )
        ) TO lt_curr.

*------------------- Posting key -------------------*
        APPEND VALUE bapiparex(
          structure  = 'POSTING_KEY'
          valuepart1 = <fs>-buzei
          valuepart2 = <fs>-bschl
        ) TO lt_ext2.

*------------------- Tax -------------------*
        IF <fs>-mwskz IS NOT INITIAL.

          APPEND VALUE bapiactx09(
            itemno_acc = <fs>-buzei
            tax_code   = <fs>-mwskz
            taxjurcode = <fs>-txjcd
            cond_key   = <fs>-kschl
            acct_key   = <fs>-ktosl
          ) TO lt_tax.

        ENDIF.

*------------------- Withholding tax -------------------*
        IF <fs>-witht IS NOT INITIAL.

          APPEND VALUE bapiacwt09(
            itemno_acc = <fs>-buzei
            wt_type    = <fs>-witht
            wt_code    = <fs>-wt_withcd
            bas_amt_lc = <fs>-wt_qsshb
            bas_amt_tc = <fs>-wt_qsshb
            man_amt_lc = <fs>-wt_qsshb
            man_amt_tc = <fs>-wt_qsshb
          ) TO lt_wt.

        ENDIF.

      ENDLOOP.

*----------------------------------------------------------------*
* 3. Simulation call (FBV0 equivalent)
*----------------------------------------------------------------*
      CALL FUNCTION 'BAPI_ACC_DOCUMENT_CHECK'
        EXPORTING
          documentheader    = ls_header
        TABLES
          accountgl         = lt_gl
          accountpayable    = lt_ap
          accountreceivable = lt_ar
          accounttax        = lt_tax
          currencyamount    = lt_curr
          extension2        = lt_ext2
          accountwt         = lt_wt
          return            = lt_return.


      et_return[] = lt_return.

    ENDIF.

  ENDMETHOD.

  METHOD display_popup.

    TYPES : BEGIN OF ty_return,
              type    TYPE bapi_mtype,
              id      TYPE symsgid,
              number  TYPE symsgno,
              message TYPE bapi_msg,
            END OF ty_return.

    DATA :  lt_return_filtered TYPE STANDARD TABLE OF ty_return.

    DATA:
      lo_alv     TYPE REF TO cl_salv_table,
      lo_columns TYPE REF TO cl_salv_columns_table,
      lo_column  TYPE REF TO cl_salv_column_table,
      lo_display TYPE REF TO cl_salv_display_settings,
      lv_message TYPE bapi_msg.

    MOVE-CORRESPONDING it_return TO lt_return_filtered[].

    LOOP AT lt_return_filtered ASSIGNING FIELD-SYMBOL(<fs_return>).
      IF <fs_return>-type = 'E' AND <fs_return>-number = '609'.
        lv_message = replace( val = <fs_return>-message
                              sub = '$'
                              with = gt_header[ is_selected ]-belnr
                              occ = 0 ) .
        <fs_return>-message = lv_message.
      ENDIF.
    ENDLOOP.


    TRY.

*------------------------------------------------------------------*
* Create SALV in popup
*------------------------------------------------------------------*
        cl_salv_table=>factory(
          IMPORTING
            r_salv_table = lo_alv
          CHANGING
            t_table      = lt_return_filtered ).


*------------------------------------------------------------------*
* Popup settings
*------------------------------------------------------------------*
        lo_alv->set_screen_popup(
          start_column = 5
          end_column   = 120
          start_line   = 3
          end_line     = 25
        ).

*------------------------------------------------------------------*
* Display settings
*------------------------------------------------------------------*
        lo_display = lo_alv->get_display_settings( ).
        lo_display->set_list_header( 'Simulation Messages (FBV0 Check)' ).
        lo_display->set_striped_pattern( abap_true ).

*------------------------------------------------------------------*
* Column handling
*------------------------------------------------------------------*
        lo_columns = lo_alv->get_columns( ).
        lo_columns->set_optimize( abap_true ).


        DATA(lo_col1) = lo_columns->get_column( 'TYPE' ).
        lo_col1->set_short_text( 'Type' ).
        lo_col1->set_medium_text( 'Severity' ).
        lo_col1->set_long_text( 'Message Type' ).

        DATA(lo_col2) = lo_columns->get_column( 'ID' ).
        lo_col2->set_medium_text( 'Msg Class' ).

        DATA(lo_col3) = lo_columns->get_column( 'NUMBER' ).
        lo_col3->set_medium_text( 'Msg No' ).

        DATA(lo_col4) = lo_columns->get_column( 'MESSAGE' ).
        lo_col4->set_medium_text( 'Message Text' ).

*------------------------------------------------------------------*
* Enable toolbar
*------------------------------------------------------------------*
        lo_alv->get_functions( )->set_all( abap_true ).

*------------------------------------------------------------------*
* Display popup
*------------------------------------------------------------------*
        lo_alv->display( ).

      CATCH cx_salv_msg cx_salv_not_found.
        RETURN.

    ENDTRY.

  ENDMETHOD.

  METHOD post_simulated_document.

    TYPES : BEGIN OF ty_items,
              belnr   TYPE belnr_d,
              bukrs   TYPE bukrs,
              gjahr   TYPE gjahr,
              bldat   TYPE bldat,
              budat   TYPE budat,
              xblnr   TYPE xblnr1,
              saknr   TYPE saknr,
              txt20   TYPE txt20_skat,
              gsber   TYPE gsber, " Business Area
              umskz   TYPE umskz, "Sepcial GL Indicator
              bupla   TYPE bupla, "Business Place
              shkzg   TYPE shkzg, "Debit/Credit Indicator
              hsn_sac TYPE j_1ig_hsn_sac, "HSN SAC Code
              dmbtr   TYPE dmbtr, "Amount in local currency
            END OF ty_items.

    DATA: lt_items_final TYPE STANDARD TABLE OF ty_items.


    DATA(lt_items) = gt_sd_fi[].

*keep only records for the selected document.
    DELETE lt_items WHERE belnr <> gt_header[ is_selected ]-belnr.

    CLEAR lt_items_final[].
*prepare table to show output.
    LOOP AT lt_items ASSIGNING FIELD-SYMBOL(<fs_items>).
      APPEND INITIAL LINE TO lt_items_final ASSIGNING FIELD-SYMBOL(<fs_items_final>).
      MOVE-CORRESPONDING <fs_items> TO <fs_items_final>.

      <fs_items_final>-dmbtr   = COND #( WHEN <fs_items_final>-shkzg = 'S'
                                 THEN  <fs_items_final>-dmbtr
                                 ELSE  (  <fs_items_final>-dmbtr * -1 ) ).

    ENDLOOP.

    IF lt_items_final IS NOT INITIAL.

      IF go_simu_post IS BOUND.
        go_simu_post->free(  ).
        FREE: go_simu_post, gr_simu_post_salv.
      ENDIF.


      TRY.

          IF go_simu_post IS INITIAL.
            CREATE OBJECT go_simu_post
              EXPORTING
                container_name = 'CC_SIMU_POST'.
          ENDIF.

          cl_salv_table=>factory(
            EXPORTING
              list_display   = if_salv_c_bool_sap=>false
              r_container    = go_simu_post
              container_name = 'CC_SIMU_POST'
           IMPORTING
              r_salv_table   = gr_simu_post_salv
            CHANGING
              t_table        = lt_items_final )."lt_items ).

*optimize column width
          gr_simu_post_salv->get_columns( )->set_optimize( abap_true ).

          DATA(lo_cols) = gr_simu_post_salv->get_columns( ).

          DATA(lo_functions) = gr_simu_post_salv->get_functions( ).
          lo_functions->set_all( abap_true ).

          lo_functions->add_function(
            name     = 'POST'
            icon     = '@9P@'
            text     = 'POST'
            tooltip  = 'Post Selected Records'
            position = if_salv_c_function_position=>right_of_salv_functions ).

**get Selections
*      gr_selections = gr_salv->get_selections( ).
*      gr_selections->set_selection_mode( cl_salv_selections=>row_column ).

**Set eventhandler.
*        DATA(lo_event_handler) = CAST lcl_event_handler( go_event_handler ).
*      SET HANDLER go_event_handler->on_user_command FOR gr_simu_post_salv->get_event( ).

*Agregation for Amount column
          DATA(lo_aggregations) = gr_simu_post_salv->get_aggregations( ).
          IF lo_aggregations IS BOUND.
            lo_aggregations->add_aggregation(
              EXPORTING
                columnname  = 'DMBTR'
                aggregation = if_salv_c_aggregation=>total ).

          ENDIF.
*data(lo_col_dmbtr) = lo_cols->get_column( 'DMBTR' ).
*lo_col_dmbtr->( if_salv_c_aggregation=>sum ).

          " 7. Display
          gr_simu_post_salv->display( ).

          CALL SCREEN 400.


        CATCH cx_salv_msg
              cx_salv_existing
              cx_salv_wrong_call
              cx_salv_not_found
              cx_salv_data_error

              INTO DATA(ls_msg2).
          MESSAGE ls_msg2->get_text( ) TYPE 'E'.
      ENDTRY.



    ENDIF.


  ENDMETHOD.

ENDCLASS.

CLASS lcl_event_handler DEFINITION.
  PUBLIC SECTION.
    METHODS:
      on_user_command FOR EVENT added_function OF cl_salv_events_table
        IMPORTING e_salv_function.

*    METHODS: on_link_click FOR EVENT link_click OF cl_salv_events_table
*      IMPORTING row column.

ENDCLASS.

CLASS lcl_event_handler IMPLEMENTATION.
  METHOD on_user_command.

    CASE e_salv_function.
      WHEN 'POST'.

**downcast global object to access internal table.
        DATA(lo_post_jv) = CAST lcl_post_jv( go_post_jv ).

*get selected rows.
        DATA(lt_selected) = gr_salv->get_selections( )->get_selected_rows( ).
        IF lt_selected IS INITIAL.
          MESSAGE 'Please select at least one document' TYPE 'I'.
          RETURN.
        ENDIF.

*Post the selected records.
        lo_post_jv->post( it_selected = lt_selected ).

      WHEN 'SIMULATE'.

        DATA : lt_return          TYPE STANDARD TABLE OF bapiret2.

*downcast global object to access internal table.
        IF lo_post_jv IS NOT BOUND.
          lo_post_jv = CAST lcl_post_jv( go_post_jv ).
        ENDIF.
*
*get selected rows.
        CLEAR lt_selected[].
        lt_selected = gr_salv->get_selections( )->get_selected_rows( ).

        IF lines( lt_selected ) GT 1.
          MESSAGE 'Please select only one document for Simulation' TYPE 'I'.
          RETURN.
        ENDIF.

*call Simulation functionality.
*        lo_post_jv->simulate_document( is_selected = lt_selected[ 1 ] ).

        lo_post_jv->simulate_document(
          EXPORTING
            is_selected = lt_selected[ 1 ]
          IMPORTING
            et_return   = lt_return
        ).

*if lt_return contains error message, then all the messages should be displayed in pop up screen.

        DATA(ls_return) = VALUE #( lt_return[ type =  'E' ] OPTIONAL ).

        IF ls_return IS NOT INITIAL.

*          lo_post_jv->display_popup( it_return = lt_return ).
          lo_post_jv->display_popup(
            EXPORTING
              is_selected = lt_selected[ 1 ]
              it_return   = lt_return
          ).

        ELSE.
          lo_post_jv->post_simulated_document( is_selected =  lt_selected[ 1 ] ).

        ENDIF.

      WHEN 'DISPLAY'.

*downcast global object to access internal table.
        IF lo_post_jv IS NOT BOUND.
          lo_post_jv = CAST lcl_post_jv( go_post_jv ).
        ENDIF.
*
*get selected rows.
        CLEAR lt_selected[].
        lt_selected = gr_salv->get_selections( )->get_selected_rows( ).

        IF lines( lt_selected ) GT 1.
          MESSAGE 'Please select only one document to display' TYPE 'I'.
          RETURN.
        ENDIF.


*Display the selected record.
        lo_post_jv->display_document( it_selected = lt_selected ).

      WHEN OTHERS.
    ENDCASE.

  ENDMETHOD.

ENDCLASS.


START-OF-SELECTION.

  TRY.
      DATA(lo_post_jv) = NEW lcl_post_jv(
                            im_bukrs = p_bukrs
                            it_budat = s_budat[]
                            it_blart = s_blart[]
                            it_belnr = s_belnr[]
                            it_user = s_user[] ).
      "im_user  = p_user ).

*store the class reference in global varaible.
      go_post_jv = lo_post_jv.

*get the data as per selection criteria.
      lo_post_jv->get_data( ).

***Display data in ALV.
*      lo_post_jv->display_data( ).

      TRY.

          IF go_main_cont IS INITIAL.
            CREATE OBJECT go_main_cont
              EXPORTING
                container_name = 'CC_MAIN'.
          ENDIF.

          cl_salv_table=>factory(
            EXPORTING
              list_display   = if_salv_c_bool_sap=>false
              r_container    = go_main_cont
              container_name = 'CC_MAIN'
           IMPORTING
              r_salv_table   = gr_salv
            CHANGING
              t_table        = lo_post_jv->gt_header[] ).

        CATCH cx_salv_msg.
          RETURN.
      ENDTRY.

*optimize column width
*      gr_salv->get_columns( )->set_optimize( abap_true ).

      DATA(lo_cols) = gr_salv->get_columns( ).



*      " 4. Add Check box Column
*      DATA(lo_columns) = gr_salv->get_columns( ).
*      lo_columns->set_optimize( abap_true ).
*      TRY.
*          DATA(lo_col) = CAST cl_salv_column_list( lo_columns->get_column( 'SEL' ) ).
*          lo_col->set_cell_type( if_salv_c_cell_type=>checkbox_hotspot ).
*          lo_col->set_short_text( 'Select' ).
*          lo_col->set_output_length( 10 ).
*        CATCH cx_salv_not_found.
*      ENDTRY.

*      " 3. Make Column Editable
*      DATA(lo_grid_api) = gr_salv->extended_grid_api( ).
*      DATA(lo_edit)     = lo_grid_api->editable_restricted( ).
*
*      lo_edit->set_attributes_for_columnname(
*        columnname = 'SEL'
*        all_cells_input_enabled = abap_true
*      ).


      "Add Custom 'SIMULATE and''POST' Button
      DATA(lo_functions) = gr_salv->get_functions( ).
      lo_functions->set_all( abap_true ).

      TRY.

*Set Company Code Column Texts.
          DATA(lo_col)  = lo_cols->get_column( 'BUKRS' ).
          lo_col->set_short_text( 'CompanCode' ).
          lo_col->set_medium_text( 'Company Code' ).
          lo_col->set_long_text( 'Company Code' ).
          lo_col->set_output_length( 12 ).
*Set Document Number Column Texts.
          DATA(lo_col1)  = lo_cols->get_column( 'BELNR' ).
          lo_col1->set_short_text( 'DocumentNo' ).
          lo_col1->set_medium_text( 'Document Number' ).
          lo_col1->set_long_text( 'Accounting Document Number' ).
          lo_col1->set_output_length( 15 ).

*Set Fiscal Year Column Texts.
          DATA(lo_col2)  = lo_cols->get_column( 'GJAHR' ).
          lo_col2->set_short_text( 'FiscalYear' ).
          lo_col2->set_medium_text( 'Fiscal Year' ).
          lo_col2->set_long_text( 'Fiscal Year' ).
          lo_col2->set_output_length( 10 ).

*Set Posting Date Column Texts.
          DATA(lo_col3)  = lo_cols->get_column( 'BUDAT' ).
          lo_col3->set_short_text( 'PostingDt' ).
          lo_col3->set_medium_text( 'Posting Date' ).
          lo_col3->set_long_text( 'Document Posting Date' ).
          lo_col3->set_output_length( 12 ).

*Set Parking Date Column Texts.(Document Date).
          DATA(lo_col4)  = lo_cols->get_column( 'BLDAT' ).
          lo_col4->set_short_text( 'DocumntDt' ).
          lo_col4->set_medium_text( 'Document Date' ).
          lo_col4->set_long_text( 'Document Date' ).
          lo_col4->set_output_length( 12 ).

*Set Reference Document Number Column Texts.
          DATA(lo_col5)  = lo_cols->get_column( 'XBLNR' ).
          lo_col5->set_short_text( 'RefDocNo' ).
          lo_col5->set_medium_text( 'Reference Doc No' ).
          lo_col5->set_long_text( 'Reference Document Number' ).
          lo_col5->set_output_length( 20 ).

*hide the user name column.
          DATA(lo_col6) = lo_cols->get_column( 'USNAM' ).
          lo_col6->set_technical( if_salv_c_bool_sap=>true ) .

*DISPLAY
          lo_functions->add_function(
            name     = 'DISPLAY'
            icon     = '@10@'
            text     = 'DISPLAY'
            tooltip  = 'Post the Selected Record'
            position = if_salv_c_function_position=>right_of_salv_functions ).

*Simulate.
          lo_functions->add_function(
            name     = 'SIMULATE'
            icon     = '@8Z@'
            text     = 'SIMULATE'
            tooltip  = 'Simulate Posting of Selected Record'
            position = if_salv_c_function_position=>right_of_salv_functions ).
*Post
          lo_functions->add_function(
            name     = 'POST'
            icon     = '@9P@'
            text     = 'POST'
            tooltip  = 'Post Selected Records'
            position = if_salv_c_function_position=>right_of_salv_functions ).
        CATCH cx_salv_existing
              cx_salv_wrong_call
              cx_salv_not_found INTO DATA(ls_msg).
          MESSAGE ls_msg->get_text( ) TYPE 'E'.
      ENDTRY.

      " 6. Register Event Handler
      DATA(lo_handler) = NEW lcl_event_handler( ).
      SET HANDLER lo_handler->on_user_command FOR gr_salv->get_event( ).

*get the eventhandler instance to use it in POST functionality at screen 400.
      go_event_handler = NEW lcl_event_handler(  ).

*get Selections
      gr_selections = gr_salv->get_selections( ).
      gr_selections->set_selection_mode( cl_salv_selections=>row_column ).

      " 7. Display
      gr_salv->display( ).

      CALL SCREEN 100.

    CATCH cx_amdp_error INTO DATA(lx_amdp).
      MESSAGE lx_amdp->get_text( ) TYPE 'E'.
    CATCH cx_salv_not_found INTO DATA(lx_not_found).
      MESSAGE lx_not_found->get_text(  ) TYPE 'E'.
  ENDTRY.

  INCLUDE zfi_post_ap_jv_status_0100o01.

  INCLUDE zfi_post_ap_jv_user_commandi01.
*********************************************************************************************************************************************************
*----------------------------------------------------------------------*
***INCLUDE ZFI_POST_AP_JV_STATUS_0100O01.
*----------------------------------------------------------------------*
*&---------------------------------------------------------------------*
*& Module STATUS_0100 OUTPUT
*&---------------------------------------------------------------------*
*&
*&---------------------------------------------------------------------*
MODULE status_0100 OUTPUT.
* SET PF-STATUS 'SALV'.
* SET TITLEBAR 'xxx'.
ENDMODULE.
*&---------------------------------------------------------------------*
*& Module STATUS_0300 OUTPUT
*&---------------------------------------------------------------------*
*&
*&---------------------------------------------------------------------*
MODULE status_0300 OUTPUT.
* SET PF-STATUS 'xxxxxxxx'.
* SET TITLEBAR 'xxx'.

  SET PF-STATUS 'PF_POST_300'.

*  IF go_msg_cont IS INITIAL.
*    CREATE OBJECT go_msg_cont
*      EXPORTING
*        container_name = 'CC_POST_MSG'.
*  ENDIF.
*
*
*    TRY.
*        cl_salv_table=>factory(
*          EXPORTING
*            r_container = go_msg_cont
*          IMPORTING
*            r_salv_table = go_msg_salv
*          CHANGING
*            t_table      = gt_msg_tab
*        ).
*
*        go_msg_salv->get_columns( )->set_optimize( abap_true ).
*        go_msg_salv->get_functions( )->set_all( abap_true ).
*        go_msg_salv->get_display_settings( )->set_list_header(
*          'Posting Result'
*        ).
*
*
*
*    CATCH cx_salv_msg.
*  ENDTRY.











ENDMODULE.
*&---------------------------------------------------------------------*
*& Module STATUS_0400 OUTPUT
*&---------------------------------------------------------------------*
*&
*&---------------------------------------------------------------------*
MODULE status_0400 OUTPUT.
  SET PF-STATUS 'PF_POST_400'.

*Set event handler.
  DATA(lo_event_handler) = CAST lcl_event_handler( go_event_handler ).
  SET HANDLER lo_event_handler->on_user_command FOR gr_simu_post_salv->get_event( ).
ENDMODULE.
*********************************************************************************************************************************************************
*----------------------------------------------------------------------*
***INCLUDE ZFI_POST_AP_JV_USER_COMMANDI01.
*----------------------------------------------------------------------*
*&---------------------------------------------------------------------*
*&      Module  USER_COMMAND_0100  INPUT
*&---------------------------------------------------------------------*
*       text
*----------------------------------------------------------------------*
MODULE user_command_0100 INPUT.

  CASE sy-ucomm.
    WHEN 'BACK' OR  '%EX'.
      LEAVE TO SCREEN 0.
    WHEN 'RW'.
      LEAVE PROGRAM.
  ENDCASE.

ENDMODULE.
*&---------------------------------------------------------------------*
*&      Module  USER_COMMAND_0300  INPUT
*&---------------------------------------------------------------------*
*       text
*----------------------------------------------------------------------*
MODULE user_command_0300 INPUT.
  CASE sy-ucomm.
    WHEN 'BACK' OR  '%EX'.
      LEAVE TO SCREEN 0.
    WHEN 'RW' or 'EXIT'.
      LEAVE PROGRAM.
  ENDCASE.

ENDMODULE.
*&---------------------------------------------------------------------*
*&      Module  USER_COMMAND_0400  INPUT
*&---------------------------------------------------------------------*
*       text
*----------------------------------------------------------------------*
MODULE user_command_0400 INPUT.


  CASE sy-ucomm.
    WHEN 'BACK' OR  '%EX'.
      LEAVE TO SCREEN 0.
    WHEN 'RW' or 'EXIT'.
      LEAVE PROGRAM.
    WHEN 'POST'.

**downcast global object to access internal table.
      DATA(lo_post_jv1) = CAST lcl_post_jv( go_post_jv ).

*get selected rows.
      DATA(lt_selected) = gr_salv->get_selections( )->get_selected_rows( ).
      IF lt_selected IS INITIAL.
        MESSAGE 'Please select at least one document' TYPE 'I'.
        RETURN.
      ENDIF.

*Post the selected records.
      lo_post_jv->post( it_selected = lt_selected ).

  ENDCASE.




ENDMODULE.
