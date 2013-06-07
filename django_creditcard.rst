==================
信用卡金流串接範例
==================

重要參數設定:
---------------

:SERVER_URL:
    本身server url
:CLIENT_ID: 金流處理公司所給予的client id 
:PAYMENT_URL: 金流處理server url
    

Form class: 定義要傳給第三方金流公司的參數
----------------------------------------------
- 包含訂單編號, client id, 金額, callback url等::

    class CreditCardConfirmForm(forms.Form):
        # 訂單編號: 當callback被呼叫時, 會傳入
        od_sob = forms.CharField(widget=HiddenInput, label=_('od_sob'))
        act = forms.CharField(widget=HiddenInput, initial="auth", label=_('act'))
        # client id
        client = forms.CharField(widget=HiddenInput, initial=settings.CLIENT_ID, label=_('client'))
        # 金額
        amount = forms.IntegerField(widget=HiddenInput, label=_('amount'))
        # callback url
        roturl = forms.CharField(widget=HiddenInput, label=_('roturl'))

View function: render form template code
------------------------------------------------------

::

    callbackUrl = "%s%s" % (settings.SERVER_URL, reverse('family_order_complete_module'))
    credit_card_form = CreditCardConfirmForm(initial={'od_sob':new_order.trans_id,
                                                      'amount':new_order.price,
                                                      'roturl': callbackUrl,
                                                      })
    return render(request,
                  'family/credit_card_payment_module.html',
                  {'credit_card_payment_url':settings.PAYMENT_URL,
                   'form': credit_card_form,
                   })


Template code: 自動送出order form
----------------------------------
 - 這個頁面其實是可以讓使用者再確認金額及商品明細等資訊, 然後再送出request到金流處理server去, 但是為了方便起見, 這裡做了auto form submit.

::

    {% load i18n %}
    <!----------------註解開始 template路徑: {{ template_name }} ---------------->
    <form method="post" id="credit_card_payment_form" action="{{credit_card_payment_url}}">
    {{ form.od_sob }}
    {{ form.act }}
    {{ form.client }}
    {{ form.amount }}
    {{ form.roturl }}
    </form>
    <script>
    	document.getElementById("credit_card_payment_form").submit();  
    </script>
    <!----------------註解結束 template路徑: {{ template_name }} ---------------->
    
Model/Form class: 用來儲存金流處理server回call時, 傳入的參數
-----------------------------------------------------------------
- Model class::

    class CreditCenterResponse(models.Model):
        client = models.CharField(max_length=6, null=True, blank=True, verbose_name=_('client'))
        succ = models.IntegerField(null=True, blank=True, verbose_name=_('succ'))
        gwsr = models.CharField(max_length=10, null=True, blank=True, verbose_name=_('gwsr'))
        response_code = models.CharField(max_length=6, null=True, blank=True, verbose_name=_('response_code'))
        response_msg = models.CharField(max_length=100, null=True, blank=True, verbose_name=_('response_msg'))
        process_date = models.CharField(max_length=8, null=True, blank=True, verbose_name=_('process_date'))
        process_time = models.CharField(max_length=6, null=True, blank=True, verbose_name=_('process_time'))
        od_sob = models.CharField(max_length=60, null=True, blank=True, verbose_name=_('od_sob'))
        auth_code = models.CharField(max_length=8, null=True, blank=True, verbose_name=_('auth_code'))
        amount = models.IntegerField(null=True, blank=True, verbose_name=_('amount'))
        od_hoho = models.CharField(max_length=500, null=True, blank=True, verbose_name=_('od_hoho'))
        eci = models.CharField(max_length=2, null=True, blank=True, verbose_name=_('eci'))
        red_dan = models.IntegerField(null=True, blank=True, verbose_name=_('red_dan'))
        red_de_amt = models.IntegerField(null=True, blank=True, verbose_name=_('red_de_amt'))
        red_ok_amt = models.IntegerField(null=True, blank=True, verbose_name=_('red_ok_amt'))
        red_yet = models.IntegerField(null=True, blank=True, verbose_name=_('red_yet'))
        rech_key = models.CharField(max_length=80, null=True, blank=True, verbose_name=_('rech_key'))
        inspect = models.CharField(max_length=60, null=True, blank=True, verbose_name=_('inspect'))
        spcheck = models.CharField(max_length=9, null=True, blank=True, verbose_name=_('spcheck'))
        card4no = models.CharField(max_length=4, null=True, blank=True, verbose_name=_('card4no'))
        card6no = models.CharField(max_length=6, null=True, blank=True, verbose_name=_('card6no'))
        expire_dt = models.CharField(max_length=4, null=True, blank=True, verbose_name=_('expire_dt'))
        inv_error = models.CharField(max_length=4, null=True, blank=True, verbose_name=_('inv_error')) 

- Form class - 以一 ModelForm 來實作::

    class CreditCenterResponseForm(forms.ModelForm):
        class Meta:
            model = CreditCenterResponse
            exclude = ('id',)
    
Callback view function 範例:
----------------------------------
- callback的流程如下:
  
  1. 先將回傳的所有POST參數寫入DB中::

        instance = form.save()
  2. 取得當初傳給金流處理server的訂單代碼, 取出訂單物件::

        od_sob = instance.od_sob
        class_trans = Class_Trans.objects.initialized_only().get(trans_id=od_sob) 

     備註: 這裡需注意金流處理server有可能對同一訂單代碼, 呼叫多次 callback function. 有可能是因為使用者還沒有看到回應畫面便refresh page. 因此這裡在做訂單query時, 要針對訂單狀態做filter.
  3. 由於需要取得使用者物件, 以顯示相關訂單訊息, 原本是從session去取得這些資訊, 但是發現有時候session 會異常被清除, 取不到任何session 資料, 因此這裡改寫, 萬一取不到session, 改由訂單代碼來取得使用者物件::

        member_id = request.session.get('member_pk') 
        if not member_id:
            member_id = class_trans.member_id
        member = Member.objects.get(id=member_id)     

- 範例::

    @csrf_exempt 
    @transaction.commit_manually 
    def order_complete_module(request):
        try:     
            #
            if request.method == 'POST': 
                form = CreditCenterResponseForm(request.POST)  
                if form.is_valid(): 
                    instance = form.save()
                    # get trans_id
                    od_sob = instance.od_sob
                    # 20130602 Ryan note:
                    # credit center may mistakenly call this function more than once, so
                    # need to add payment_status filter
                    # only status with 'initialized' can be selected
                    class_trans = Class_Trans.objects.initialized_only().select_related("klass").get(trans_id=od_sob) 
                    member_id = request.session.get('member_pk') 
                    if not member_id:
                        member_id = class_trans.member_id
                    member = Member.objects.get(id=member_id)            
                    #
                    succ = instance.succ
                    if succ:
                        class_trans.payment_status = Class_Trans.PAYMENT_STATUS_CODE['COMPLETE']
                        class_trans.trans_complete_date = timezone.now()
                        class_trans.save()
                        klass = class_trans.klass
                        # update klass confirmed_register_count
                        if class_trans.member_status_code == MEMBER_STATUS_CODE['GROUP']:
                            klass.confirmed_register_count = F('confirmed_register_count') + 2
                        else:
                            klass.confirmed_register_count = F('confirmed_register_count') + 1
                        klass.save()
                        # send a confirm email
                        confirm_payment_email_ok = _family_send_confirm_payment_email(member, klass, class_trans, _("payment_type_credit_card"))
                        class_trans.confirm_payment_email_ok = confirm_payment_email_ok
                        class_trans.save()
                        #
                        survey_url = class_trans.get_survey_url()
                        message = _("Congratulation! Your registration and payment have completed!")
                        transaction.commit()
                        return render(request, 'family/order_complete.html', {'success':succ,
                                                                              'message':message,
                                                                              'klass': klass,
                                                                              'register_index':class_trans.register_index,
                                                                              'survey_url':survey_url,
                                                                              'family_url': settings.FAMILY_WEB_SITE_URL})
                    else:
                        class_trans.payment_status = Class_Trans.PAYMENT_STATUS_CODE['FAILED'] 
                        class_trans.save()
                        response_msg = instance.response_msg
                        message = _("Your payment has failed. Error message: %(error_msg)s") % {'error_msg':response_msg} 
                        transaction.commit()
                        return render(request, 'family/order_complete.html', {'success':succ, 'message':message, 'family_url': settings.FAMILY_WEB_SITE_URL})
                else:
                    raise Exception
            else:
                # doen't support GET   
                # not quite sure why it keeps being called
                # log GET parameters
                if request.GET:
                    for key, value in request.GET.iteritems() :
                        print "key=%s, value=%s" % (key, value)                   
                raise Exception
        except: 
            transaction.rollback()
            error_id = unique_id() 
            logger.warning("Error when calling %s: error_id=%s" % (sys._getframe().f_code.co_name, error_id), exc_info=1)        
            message = _("Unexpected Error: error_id=%(error_id)s") % {'error_id': error_id} 
            status_code = 500 
            return render(request, "500.html", dictionary={'message':message, }, status=status_code)
