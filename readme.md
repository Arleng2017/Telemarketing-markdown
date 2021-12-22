#  Telemarketing
Gitlab Repo 

[Repo Link](http://gitlab.bla.co.th/telemarketing/telemarketing.api)
> http://gitlab.bla.co.th/telemarketing/telemarketing.api
        
💻 หน้า UI หลัก 

<img width="949" alt="main desktop" src="https://user-images.githubusercontent.com/46476206/147031685-0121d171-49b2-4d56-a437-488bb3786d69.png">


---
## ⚙️ **หลักการทำงานของโค้ด**

## 📌 **Upload**
---
### 🎬 **First Billing** 
🔎 รายการส่งตัดบัตรเครดิตงวดแรก


### 🎬 **Yes Sale** 
🔎 เพื่อจัดส่งรายการ Yes sale ที่ตรวจสอบคุณภาพเรียบร้อยแล้วให้ BLA พิจารณาออกกรมธรรม์

<img width="959" alt="yessale" src="https://user-images.githubusercontent.com/46476206/147037281-a741cc06-fc60-4cbb-a14e-ccc6f9e10844.png">

*การทำงานของ Code*
``` c#
[HttpPost]
[Route("UploadYesSale")]
[Authorize(Roles = "Tele.IDBL.UploadYesSale, Tele.TVD.UploadYesSale")]
public IHttpActionResult UploadYesSale(FileInfo fileInfo)
{
    try
    {
        // step 1 เช็คว่าเป็น Broker ที่กำหนดไว้หรือไม่?
        // แต่ก่อนมี TVD เป็น Broker แต่ตอนนี้ได้ยกเลิกไปแล้ว
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }

        ObjectParam param = new ObjectParam()
        {
            User = user,
            File = fileInfo
        };

        ITeleServiceAction action = new TeleServiceAction();
        
        // step 2 ก็มาทำที่ Bussiness logic แล้วจะเอาข้อมูลไฟล์ที่ได้ไป Save เพื่อไปบันทึกลง Database
        action.UploadYesSale(param);
        return Ok();
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}

```


```c#
public void UploadYesSale(ObjectParam param)
{
    // step 1 ไปดึงข้อมูลของแต่ละไฟล์ออกมา ว่ามีกี่รายการ/ แปลงข้อมูลจากไป เป็น Item
    DelimitedFileEngine engine = new DelimitedFileEngine(typeof(YesSaleLayout));
    ITeleRepository repository = new TeleRepository();
    IEnumerable<YesSaleLayout> items = null;

    string strContentFile = Encoding.GetEncoding(874).GetString(param.File.Content);
    try
    {
        //อ่านค่า Content แปลงฟอร์แมตออกมา
        items = engine.ReadString(strContentFile) as IEnumerable<YesSaleLayout>;
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }
        throw new ApplicationException("ไฟล์ข้อมูลไม่ถูกต้อง กรุณาตรวจสอบไฟล์ : " + e);
    }

    //step 2. เมื่ออ่านค่ามาแล้ว ก็จะส่งข้อมูลที่ได้ไปยัง Database 
    repository.UploadYesSale(param, items);
    
    // step3. send mail
    DateTime now = repository.GetCurrentDateTime();
    var variables = new Dictionary<string, string>() {
        { "Function", "Upload Yes Sale File" },
        { "RunDateTime", now.ToLongDateString() + ' ' + now.ToLongTimeString() },
        { "TotalItem", string.Format("{0:N0}", items.Count()) }
    };

    string mailCode = string.Empty;
    string partyCode = param.User.Company;
    if (partyCode == PartyCode.IDBL)
    {
        mailCode = MailCode.TELE_IDBL;
    }
    else if (partyCode == PartyCode.TVD)
    {
        mailCode = MailCode.TELE_TVD;
    }

    EMailFromDb email = new EMailFromDb();
    try
    {
        email.Send(mailCode, variables);
    }
    catch (Exception ex)
    {
        throw new ApplicationException("ไม่สามารถส่ง E-mail ได้ : " + ex.Message);
    }
}

```
### 🎬 **Cancel Case** 
🔎 เพื่อจัดส่งรายการลูกค้าที่ได้รับการติดต่อจาก Confirmation call แต่มีความประสงค์จะยกเลิกกธ. ให้ BLA

<img width="959" alt="Cancel Case" src="https://user-images.githubusercontent.com/46476206/147037350-49db2bce-19c9-4aa5-a7f2-9769ba5b5151.png">

*การทำงานของ Code*
``` c#
[HttpPost]
[Route("UploadCancelCase")]
[Authorize(Roles = "Tele.IDBL.UploadCancel, Tele.TVD.UploadCancel")]
public IHttpActionResult UploadCancelCase(FileInfo fileInfo)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }

        ObjectParam param = new ObjectParam()
        {
            User = user,
            File = fileInfo
        };

        ITeleServiceAction action = new TeleServiceAction();
        action.UploadCancelCase(param);
        return Ok();
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Application Info** 
🔎 เพื่อส่งข้อมูลคำขอเอาประกันภัย สำหรับรายการชำระผ่านช่องทาง Counter Service (7-11)

<img width="956" alt="ApplicationInfo" src="https://user-images.githubusercontent.com/46476206/147037423-5bd37024-414e-4087-a8df-234014f68cc3.png">



*การทำงานของ Code*
``` c#
 [HttpPost]
        [Route("UploadApplicationInfo")]
        [Authorize(Roles = "Tele.IDBL.UploadApplicationInfo, Tele.TVD.UploadApplicationInfo")]
        public IHttpActionResult UploadApplicationInfo(FileInfo fileInfo)
        {
            try
            {
                var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
                if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
                {
                    throw new UnauthorizedAccessException("Invalid Company");
                }

                ObjectParam param = new ObjectParam()
                {
                    User = user,
                    File = fileInfo
                };

                ITeleServiceAction action = new TeleServiceAction();
                action.UploadApplicationInfo(param);
                return Ok();
            }
            catch (UnauthorizedAccessException ex)
            {
                return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
            }
            catch (ApplicationException ex)
            {
                return BadRequest(ex.Message);
            }
            catch (Exception ex)
            {
                string e = ex.Message;
                if (ex.InnerException != null)
                {
                    e = e + " ==> " + ex.InnerException.Message;
                    if (ex.InnerException.InnerException != null)
                    {
                        e = e + " ==> " + ex.InnerException.InnerException.Message;
                    }
                }

                Logger logger = LogManager.GetCurrentClassLogger();
                logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
                return InternalServerError(ex);
            }
        }

```

## 📌 **Download**
---
### 🎬 **First Billing** 
🔎 ผลการตัดบัตรเครดิตงวดแรก

<img width="955" alt="DownloadFirstBilling" src="https://user-images.githubusercontent.com/46476206/147037476-f0cb2b8c-5eb3-4385-a437-a2fa65488614.png">

*การทำงานของ Code*
``` c#

```
### 🎬 **Policy Update** 
🔎 เพื่อ BLA จัดส่งรายการลูกค้าที่ออกกรมธรรม์แล้ว ให้ IDB โดยระบุ Policy no. Mailing date เพื่อใช้ติดต่อทำ Confirmation call 
การส่งข้อมูล จัดส่งเฉพาะรายการที่ส่งกธ.ให้ลูกค้าแล้ว ตาม Transaction date"

<img width="961" alt="PolicyUpdate" src="https://user-images.githubusercontent.com/46476206/147037523-24482573-4cd2-405e-8eff-601c4caa4cf9.png">


*การทำงานของ Code*

``` c#
[HttpPost]
[Route("DownloadPolicyUpdate")]
[Authorize(Roles = "Tele.IDBL.DownloadPolicyUpdate, Tele.TVD.DownloadPolicyUpdate")]
public IHttpActionResult DownloadPolicyUpdate(DownloadFileInfo downloadFileInfo)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        ITeleServiceAction action = new TeleServiceAction();
        var result = action.DownloadPolicyUpdateFile(user, downloadFileInfo.ContentId);
        return Ok(result);
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Policy Cancel** 
🔎 เพื่อ BLA จัดส่งรายการยกเลิกกธ. ให้ IDB update Policy status

<img width="960" alt="Policy Cancel" src="https://user-images.githubusercontent.com/46476206/147037595-e8582ca2-c4c3-44be-9f1d-f6a094ea5306.png">


*การทำงานของ Code*
``` c#
[HttpPost]
[Route("DownloadPolicyCancel")]
[Authorize(Roles = "Tele.IDBL.DownloadPolicyCancel, Tele.TVD.DownloadPolicyCancel")]
public IHttpActionResult DownloadPolicyCancel(DownloadFileInfo downloadFileInfo)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        ITeleServiceAction action = new TeleServiceAction();
        var result = action.DownloadPolicyCancelFile(user, downloadFileInfo.ContentId);
        return Ok(result);
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Payment Confirmation** 
🔎 เพื่อส่งข้อมูลยืนยันการได้รับชำระเงินเบี้ยประกันภัย สำหรับรายการชำระผ่านช่องทาง Counter Service (7-11)

<img width="959" alt="Payment Confirimation" src="https://user-images.githubusercontent.com/46476206/147043060-36012a37-7f84-4f84-a192-e9fff09513eb.png">



*การทำงานของ Code*
``` c#
[HttpPost]
[Route("DownloadPaymentConfirmation")]
[Authorize(Roles = "Tele.IDBL.DownloadPaymentConfirmation, Tele.TVD.DownloadPaymentConfirmation")]
public IHttpActionResult DownloadPaymentConfirmation(DownloadFileInfo downloadFileInfo)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        ITeleServiceAction action = new TeleServiceAction();
        var result = action.DownloadPaymentConfirmationFile(user, downloadFileInfo.ContentId);                
        return Ok(result);
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Paycode Follow Up** 
🔎 ติดตามรายการ paycode ที่ยังไม่ได้ชำระเงิน

<img width="959" alt="Paycode Follow up" src="https://user-images.githubusercontent.com/46476206/147043109-21dc5ee2-ce08-4c7c-a3c8-a420f5bd2611.png">


*การทำงานของ Code*
``` c#
[HttpPost]
[Route("DownloadPaycodeFollowUp")]
[Authorize(Roles = "Tele.IDBL.DownloadPaycodeFollowUp, Tele.TVD.DownloadPaycodeFollowUp")]
public IHttpActionResult DownloadPaycodeFollowUp(PaycodeFollowupParam paycodeFollowDate)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        IFollowUpAction action = new FollowUpAction();
        var results = action.CreatedNewPaycodeFollowUp(user, paycodeFollowDate);
        if(results == null)
        {
            return NotFound();
        }

        return Ok(results);
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ArgumentException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Paycode Reply** 
🔎 รายการที่ทำการตอบกลับไปทาง I-Direct เพื่อให้ทาง I-Direct Download ไปติดตามลูกค้า

<img width="956" alt="Paycode Reply" src="https://user-images.githubusercontent.com/46476206/147043128-09f989bf-e689-46b7-b354-1df02850edfa.png">

*การทำงานของ Code*
``` c#
[HttpPost]
[Route("DownloadPaycodeReply")]
[Authorize(Roles = "Tele.IDBL.DownloadPaycodeReply, Tele.TVD.DownloadPaycodeReply")]
public IHttpActionResult DownloadPaycodeReply(DownloadFileInfo downloadFileInfo)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        ITeleServiceAction action = new TeleServiceAction();
        var result = action.DownloadPaycodeReplyFile(user, downloadFileInfo.ContentId);
        return Ok(result);
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Recurring Follow Up** 
🔎 ติดตามรายการตัดบัตรที่ยังไม่ได้ชำระเงิน

<img width="958" alt="Recurring Follow UP" src="https://user-images.githubusercontent.com/46476206/147043176-3b9bd2ca-8762-4cac-8775-1160a03fe604.png">

*การทำงานของ Code*
``` c#
[HttpPost]
[Route("DownloadRecurringFollowUp")]
[Authorize(Roles = "Tele.IDBL.DownloadRecurringFollowUp, Tele.TVD.DownloadRecurringFollowUp")]
public IHttpActionResult DownloadRecurringFollowUp()
{
    try
    {
        string recurringFollowUpUrl = ConfigurationManager.AppSettings.Get("RecurringFollowUpUrl");

        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL && user.Company != PartyCode.TVD)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        IFollowUpAction commonAction = new FollowUpAction();
        //string partyCode = commonAction.GetPartyCode(userInfo); ;
        string partyCode = user.Company;

        using (HttpClient client = new HttpClient())
        {
            HttpResponseMessage response = client.GetAsync(recurringFollowUpUrl + "LoadFollowUpReport/" + partyCode).Result;
            FileData resultFile = response.Content.ReadAsAsync<FileData>().Result;
            if (resultFile == null)
            {
                return NotFound();
            }
            FileInfo result = new FileInfo();
            result.Filename = resultFile.FileName;
            result.Content = resultFile.Content;

            /// TODO : Send Mail  FollowUp Recurring
            /// // resultFile.ItemCount
            IFollowUpAction action = new FollowUpAction();
            action.SendMail(partyCode, resultFile.ItemCount, "Download Recurring FollowUp File");

            return Ok(result);
        }
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ArgumentException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```
### 🎬 **Sale Lead** 
🔎 เพื่อ BLA จัดส่งรายการ DRTV's Call List

<img width="956" alt="sale lead" src="https://user-images.githubusercontent.com/46476206/147043215-32c5b962-7018-4663-8724-4a9b97037bbb.png">

*การทำงานของ Code*
``` c#
[HttpPost]
[Route("DownloadSaleLead")]
[Authorize(Roles = "Tele.IDBL.DownloadSaleLead")]
public IHttpActionResult DownloadSaleLead(DownloadFileInfo downloadFileInfo)
{
    try
    {
        var user = ApplicationInfoProvider.GetUserInfo(User.Identity as ClaimsIdentity);
        if (user.Company != PartyCode.IDBL)
        {
            throw new UnauthorizedAccessException("Invalid Company");
        }
        ITeleServiceAction action = new TeleServiceAction();
        var result = action.DownloadSaleLeadFile(user, downloadFileInfo.ContentId);
        return Ok(result);
    }
    catch (UnauthorizedAccessException ex)
    {
        return Content(HttpStatusCode.Unauthorized, new ResponseMessage() { Message = ex.Message });
    }
    catch (ApplicationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        string e = ex.Message;
        if (ex.InnerException != null)
        {
            e = e + " ==> " + ex.InnerException.Message;
            if (ex.InnerException.InnerException != null)
            {
                e = e + " ==> " + ex.InnerException.InnerException.Message;
            }
        }

        Logger logger = LogManager.GetCurrentClassLogger();
        logger.Error("MethodName : " + MethodBase.GetCurrentMethod().Name + " ==> " + e);
        return InternalServerError(ex);
    }
}
```

