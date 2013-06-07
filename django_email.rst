=================
Django Email 範例 
=================

settings.py
------------
- For hinet smtp setting::

    EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend' 
    EMAIL_HOST = "msa.hinet.net"
    EMAIL_PORT = 25
    

- For google smtp setting::

    EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend' 
    EMAIL_HOST = "smtp.gmail.com"
    EMAIL_PORT = 587
    EMAIL_HOST_USER = "your gmail account"
    EMAIL_HOST_PASSWORD = "your password"
    EMAIL_USE_TLS = True

- for testing only, ie. the email content displays in console, instead of actually sending the email::
    
    EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

email python code
-----------------
- Use EmailMessage class::

    msg_subject = "your email subject"      
    # Email subject *must not* contain newlines
    msg_subject = ''.join(msg_subject.splitlines())
    msg_body = "your email body" 
    from_email = 'from_email_here'
    recipient_list = []
    recipient_list.append('recipient_email_here') 
    # use EmailMessage class
    msg = EmailMessage(msg_subject, msg_body, from_email, recipient_list)
    # if there is attachment 
    msg.attach_file('your attachment file path')
    # if main content need to be in text/html
    msg.content_subtype = "html" 
    msg.send()

- Use user.email_user() in django auth module::

    user.email_user(subject, message, 'from_email_here')
        


