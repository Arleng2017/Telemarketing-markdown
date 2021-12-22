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

 > save ข้อมูลลง db

 > สร้างไฟล์สำหรับส่งตัดบัตร

 > วางไฟล์ไปที่  sftp server เพื่อตัดบัตร"

🧪 **source**

    isishq11.bla.co.th/ISIS2013/ISIS.Telemarketing

### 🎬 **Yes Sale** 
🔎 เพื่อจัดส่งรายการ Yes sale ที่ตรวจสอบคุณภาพเรียบร้อยแล้วให้ BLA พิจารณาออกกรมธรรม์

<img width="959" alt="yessale" src="https://user-images.githubusercontent.com/46476206/147037281-a741cc06-fc60-4cbb-a14e-ccc6f9e10844.png">

*การทำงานของ Code*

🔑 **UploadYesSale**
> step 1 เช็คว่าเป็น Broker ที่กำหนดไว้หรือไม่?

> step 2 ก็มาทำที่ Bussiness logic แล้วจะเอาข้อมูลไฟล์ที่ได้ไป Save

``` c#
[HttpPost]
[Route("UploadYesSale")]
[Authorize(Roles = "Tele.IDBL.UploadYesSale, Tele.TVD.UploadYesSale")]
public IHttpActionResult UploadYesSale(FileInfo fileInfo)
{
    try
    {
        💡 // step 1 เช็คว่าเป็น Broker ที่กำหนดไว้หรือไม่?
        💡 // แต่ก่อนมี TVD เป็น Broker แต่ตอนนี้ได้ยกเลิกไปแล้ว
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
        
        💡 // step 2 ก็มาทำที่ Bussiness logic แล้วจะเอาข้อมูลไฟล์ที่ได้ไป Save เพื่อไปบันทึกลง Database
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
**UploadYesSale**
> step 1. ไปดึงข้อมูลของแต่ละไฟล์ออกมา ว่ามีกี่รายการ/ แปลงข้อมูลไปเป็น Item

> step 2. เมื่ออ่านค่ามาแล้ว ก็จะส่งข้อมูลที่ได้ไปยัง Database 

> step 3. send mail

```c#
public void UploadYesSale(ObjectParam param)
{
    // step 1 ไปดึงข้อมูลของแต่ละไฟล์ออกมา ว่ามีกี่รายการ/ แปลงข้อมูลไป เป็น Item
    DelimitedFileEngine engine = new DelimitedFileEngine(typeof(YesSaleLayout));
    ITeleRepository repository = new TeleRepository();
    IEnumerable<YesSaleLayout> items = null;

    string strContentFile = Encoding.GetEncoding(874).GetString(param.File.Content);
    try
    {
        💡 //อ่านค่า Content แปลงฟอร์แมตออกมา
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

    💡 //step 2. เมื่ออ่านค่ามาแล้ว ก็จะส่งข้อมูลที่ได้ไปยัง Database 
    repository.UploadYesSale(param, items);
    
    💡 //  step3. send mail
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


👨🏻‍💻 **การทำงานของ Code** 

🔑 **UploadCancelCase**

> step 1. Upload รายการที่ลูกค้าก่อนยกเลิก

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

        💡 //step 1. Upload รายการลูกค้าก่อนยกเลิก
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

**UploadCancelCase**

> สร้าง Object เพื่อมา Assign ค่า

> ระบบส่งรายการที่มีการยกเลิก เข้าสู่ระบบ NewBis

> บันทึกข้อมูลรายการลง Database

> ถ้าทำรายการก่อนหน้าเสร็จสิ้น ก็จะมาส่ง Email

```c#
   public void UploadCancelCase(ObjectParam param)
{
    DelimitedFileEngine engine = new DelimitedFileEngine(typeof(CancelCaseLayout));
    ITeleRepository repository = new TeleRepository();
    IEnumerable<CancelCaseLayout> items = null;

    string strContentFile = Encoding.GetEncoding(874).GetString(param.File.Content);
    try
    {
        items = engine.ReadString(strContentFile) as IEnumerable<CancelCaseLayout>;
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

    #region save data to database & call policy service
    string cancelCaseId = repository.GetNewID();
    string contentId = repository.GetNewID();
    DateTime now = repository.GetCurrentDateTime();
    string username = param.User.Username;
    string partyCode = param.User.Company;

    💡 // สร้าง Object เพื่อมา Assign ค่า
    IDL_SALE_CONTENT content = new IDL_SALE_CONTENT();
    content.CONTENT_ID = contentId;
    content.FILE_NAME = param.File.Filename;
    content.CONTENT = param.File.Content;
    content.CONTENT_FILE_TYPE = ContentFileType.CancelCase;
    content.TELE_SALE_CONTENT_OWNER = new TELE_SALE_CONTENT_OWNER();
    content.TELE_SALE_CONTENT_OWNER.TRD_PARTY_ID = repository.GetPartyID(partyCode);
    content.TELE_SALE_CONTENT_OWNER.CREATED_ON = now;

    IDL_CANCEL_CASE cancelCase = new IDL_CANCEL_CASE();
    cancelCase.CANCEL_CASE_ID = cancelCaseId;
    cancelCase.CONTENT_ID = contentId;
    cancelCase.CREATED_ON = now;
    cancelCase.CREATED_BY = username;

    using (I_DirectSvcClient client = new I_DirectSvcClient())
    {
        int seqNumber = 0;
        foreach (var item in items)
        {
            seqNumber++;

            IDL_CANCEL_CASE_ITEM cancelItem = new IDL_CANCEL_CASE_ITEM();
            cancelItem.SALE_ITEM_ID = repository.GetSaleItemIDFromRefNo(item.ReferenceNumber);
            cancelItem.CANCEL_CASE_ID = cancelCaseId;
            cancelItem.SEQ_NUMBER = seqNumber;
            cancelItem.CANCEL_CASE_DATE = DateTime.ParseExact(item.CancelCaseDate, "yyyyMMdd", CultureInfo.InvariantCulture);
            cancelItem.CANCEL_TRANSACTION_DATE = DateTime.ParseExact(item.CancelCaseTransactionDate, "yyyyMMdd", CultureInfo.InvariantCulture);
            cancelCase.IDL_CANCEL_CASE_ITEM.Add(cancelItem);

            💡 //send cancel info to policy service
            💡 // ระบบส่งรายการที่มีการยกเลิก เข้าสู่ระบบ NewBis
            var result =  client.RequestToCancelPolicy(item.PolicyNumber, DateTime.ParseExact(item.CancelCaseDate, "yyyyMMdd", CultureInfo.InvariantCulture));
            if (!result.Successed)
            {
                throw new ApplicationException("ไม่สามารถส่งข้อมูลให้ Policy Service ได้ : "+result.ErrorMessage);
            }
        } 
    }

    content.IDL_CANCEL_CASE.Add(cancelCase);
    💡 //บันทึกข้อมูลรายการลง Database
    repository.SavePolicyCancel(content);
    #endregion
    
    💡 // send mail
    💡 //ถ้าทำรายการก่อนหน้าเสร็จสิ้น ก็จะมาส่ง Email
    var variables = new Dictionary<string, string>() {
        { "Function", "Upload Cancel Case File" },
        { "RunDateTime", now.ToLongDateString() + ' ' + now.ToLongTimeString() },
        { "TotalItem", string.Format("{0:N0}", items.Count()) }
    };

    string mailCode = string.Empty;
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

**SavePolicyCancel**

> บันทึกข้อมูลรายการลง Database

```c# 
 public void SavePolicyCancel(IDL_SALE_CONTENT saleContent)
        {
            using (TeleEntities context = new TeleEntities())
            {
                context.IDL_SALE_CONTENT.Add(saleContent);
                try
                {
                    context.SaveChanges();
                }
                catch (DbEntityValidationException dbEx)
                {
                    var errorMessages = dbEx.EntityValidationErrors
                        .SelectMany(x => x.ValidationErrors)
                        .Select(x => x.ErrorMessage);

                    var fullErrorMessage = string.Join("; ", errorMessages);
                    var exceptionMessage = string.Concat(dbEx.Message, " The validation errors are: ", fullErrorMessage);
                    throw new Exception("=== Error!!! Method Name : " + MethodBase.GetCurrentMethod().Name + " ===\n" + exceptionMessage);
                }
            }
        }

```
### 🎬 **Application Info** 
🔎 เพื่อส่งข้อมูลคำขอเอาประกันภัย สำหรับรายการชำระผ่านช่องทาง Counter Service (7-11)

<img width="956" alt="ApplicationInfo" src="https://user-images.githubusercontent.com/46476206/147037423-5bd37024-414e-4087-a8df-234014f68cc3.png">


👨🏻‍💻 **การทำงานของ Code**

🔑 **UploadApplicationInfo**

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

                // 💡

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

**UploadApplicationInfo**

```c#
 public void UploadApplicationInfo(ObjectParam param)
{
    DelimitedFileEngine engine = new DelimitedFileEngine(typeof(ApplicationInfoLayout));
    ITeleRepository repository = new TeleRepository();
    DateTime now = repository.GetCurrentDateTime();
    IEnumerable<ApplicationInfoLayout> items = null;

    string strContentFile = Encoding.GetEncoding(874).GetString(param.File.Content);
    try
    {
        items = engine.ReadString(strContentFile) as IEnumerable<ApplicationInfoLayout>;
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

    repository.UploadApplicationInfo(param, items);

    #region Send SMS
    List<string> messages = new List<string>();
    List<IDL1_APP_INFO_SEND_SMS> appInfoSMSList = new List<IDL1_APP_INFO_SEND_SMS>();

    var csoLinkIdlFypList = repository.GetAllItemToSendSMS();
    foreach (var csoLinkIdlFyp in csoLinkIdlFypList)
    {
        IDL1_APP_INFO_SEND_SMS appInfoSMS = new IDL1_APP_INFO_SEND_SMS();
        appInfoSMS.APP_INFO_ITEM_ID = csoLinkIdlFyp.APP_INFO_ITEM_ID;
        appInfoSMS.MESSAGE = "ชำระค่าเบี้ยประกันที่ 7-11 paycode = " + csoLinkIdlFyp.CSO_PAY_ITEM.CS_PAY_CODE + " จำนวน " +
            csoLinkIdlFyp.CSO_PAY_ITEM.AMOUNT.ToString("N") + " บาท สอบถามเพิ่มเติมโทร 02-777-8888 กรุงเทพประกันชีวิต";
        appInfoSMS.MOBILE_TEL = csoLinkIdlFyp.IDL1_APP_INFO_ITEM.MOBILE_TEL1;
        appInfoSMS.CREATED_ON = now;
        appInfoSMSList.Add(appInfoSMS);

        string msg = appInfoSMS.MOBILE_TEL + ",'" + appInfoSMS.MESSAGE + "'";
        messages.Add(msg);
    }
    #endregion

    #region เพิ่มการส่ง SMS ให้ผู้ดูแลระบบ
    string sendMessage = "วันที่ " + now.ToString("dd/MM/yyyy HH:mm:ss") + " ทำการ Upload Application Info จำนวน " + messages.Count.ToString() + " รายการ";
    var smsReceivers = ConfigurationManager.AppSettings["SMSReceivers"];
    List<string> telNoList = new List<string>();
    if (smsReceivers.Contains(","))
    {
        telNoList = smsReceivers.Split(',').ToList();
    }
    else
    {
        telNoList.Add(smsReceivers);
    }
    foreach (var telNo in telNoList)
    {
        messages.Add(telNo + ",'" + sendMessage + "'");
    }
    #endregion

   💡 // Send SMS
    bool result = false;
    result = CallServiceSendSMS(messages);
    if (result)
    {
        repository.SaveAppInfoSendSMS(appInfoSMSList);

        💡 // send mail
        var variables = new Dictionary<string, string>() {
            { "Function", "Upload Application Info File" },
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
    else
    {
        throw new ApplicationException("ไม่สามารถส่ง SMS ให้ลูกค้าได้");
    }
}
```
**UploadApplicationInfo**

> นำข้อมูลที่ได้มาวนลูป เพื่อ Save ลง Databse

> ทำการ Gen PayCode แล้วก็บันทึกข้อมูลเพื่อรอส่ง SMS

```c# 
 public void UploadApplicationInfo(ObjectParam param, IEnumerable<ApplicationInfoLayout> items)
{
    using (TeleEntities context = new TeleEntities())
    {
        #region  Insert data to IDL1_App_Info_Item
        DateTime now = GetCurrentDbDataTime();
        string contentId = GetNewDbID();
        int seqNumber = 0;
        string username = param.User.Username;
        💡 //string partyCode = GetPartyCode(param.User);
        string partyCode = param.User.Company;
        IDL_SALE_CONTENT content = new IDL_SALE_CONTENT();
        content.CONTENT_ID = contentId;
        content.FILE_NAME = param.File.Filename;
        content.CONTENT = param.File.Content;
        content.CONTENT_FILE_TYPE = ContentFileType.ApplicationInfo;
        
        💡 //นำข้อมูลที่ได้มาวนลูป เพื่อ Save ลง Databse
        foreach (var item in items)
        {
            seqNumber++;
            string appInfoId = GetNewDbID();
            string appInfoItemId = GetNewDbID();

            IDL1_APP_INFO appInfo = new IDL1_APP_INFO();
            appInfo.APP_INFO_ID = appInfoId;
            appInfo.CONTENT_ID = contentId;
            appInfo.APP_INFO_ITEM_ID = appInfoItemId;

            appInfo.IDL1_APP_INFO_ITEM = new IDL1_APP_INFO_ITEM();
            appInfo.IDL1_APP_INFO_ITEM.APP_INFO_ITEM_ID = appInfoItemId;
            appInfo.IDL1_APP_INFO_ITEM.TRANSACTION_DATE = DataConverter.ConvertStringToDateTime(item.TransactionDate);
            appInfo.IDL1_APP_INFO_ITEM.REF_NO = item.RefNo;
            appInfo.IDL1_APP_INFO_ITEM.SEQ_NUMBER = seqNumber;
            appInfo.IDL1_APP_INFO_ITEM.INSURED_TITLE = item.InsuredTitle;
            appInfo.IDL1_APP_INFO_ITEM.INSURED_NAME = item.InsuredName;
            appInfo.IDL1_APP_INFO_ITEM.INSURED_SURNAME = item.InsuredSurname;
            appInfo.IDL1_APP_INFO_ITEM.ID_TYPE = item.IDType;
            appInfo.IDL1_APP_INFO_ITEM.ID_CARD = item.IDCard;
            appInfo.IDL1_APP_INFO_ITEM.MOBILE_TEL1 = item.MobileTel1;
            appInfo.IDL1_APP_INFO_ITEM.CAMPAIGN_CODE = item.CampaignCode;
            appInfo.IDL1_APP_INFO_ITEM.PRODUCT_CODE = item.ProductCode;
            appInfo.IDL1_APP_INFO_ITEM.PRODUCT_SUBCODE = item.ProductSubCode;
            appInfo.IDL1_APP_INFO_ITEM.PRODUCT_PLAN = item.ProductPlan;
            appInfo.IDL1_APP_INFO_ITEM.BILL_MODE = item.BillMode;
            appInfo.IDL1_APP_INFO_ITEM.TOTAL_SALE_PREMIUM = Convert.ToDecimal(item.TotalSalePremium);
            appInfo.IDL1_APP_INFO_ITEM.CREATED_ON = now;
            appInfo.IDL1_APP_INFO_ITEM.CREATED_BY = username;

            // add 3rd party
            appInfo.IDL1_APP_INFO_ITEM.IDL8_APP_INFO_ITEM_OWNER = new IDL8_APP_INFO_ITEM_OWNER();
            appInfo.IDL1_APP_INFO_ITEM.IDL8_APP_INFO_ITEM_OWNER.APP_INFO_ITEM_ID = appInfoItemId;
            appInfo.IDL1_APP_INFO_ITEM.IDL8_APP_INFO_ITEM_OWNER.TRD_PARTY_ID = GetPartyIdLocal(partyCode);
            appInfo.IDL1_APP_INFO_ITEM.IDL8_APP_INFO_ITEM_OWNER.CREATED_ON = now;
            
            content.IDL1_APP_INFO.Add(appInfo);
        }

        context.IDL_SALE_CONTENT.Add(content);
        try
        {
            context.SaveChanges();
        }
        catch (DbEntityValidationException dbEx)
        {
            var errorMessages = dbEx.EntityValidationErrors
                .SelectMany(x => x.ValidationErrors)
                .Select(x => x.ErrorMessage);

            var fullErrorMessage = string.Join("; ", errorMessages);
            var exceptionMessage = string.Concat(dbEx.Message, " The validation errors are: ", fullErrorMessage);
            throw new Exception("=== Error!!! Method Name : " + MethodBase.GetCurrentMethod().Name + " Line: " + seqNumber + " ===\n" + exceptionMessage);
        }
        #endregion

        💡 // ทำการ Gen PayCode แล้วก็บันทึกข้อมูลเพื่อรอส่ง SMS
        #region Call package for insert data into CSO_Pay_item and CSO_Link_IDL_FYP
        string connectionString = ConfigurationManager.ConnectionStrings["isis_db"].ConnectionString;
        using (OracleConnection con = new OracleConnection(connectionString))
        {
            con.Open();
            string command = "cso_idl_fyp.gen_idl_fyp_pay_item";
            OracleCommand cmd = new OracleCommand(command, con);
            cmd.CommandType = CommandType.StoredProcedure;
            cmd.ExecuteNonQuery();
        }
        #endregion

         // created paycode reply file
        SavePaycodeReply(partyCode, username);
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

🔑 **DownloadPolicyUpdate**
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


👨🏻‍💻 **การทำงานของ Code**

🔑 **DownloadPolicyCancel**
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

 **DownloadPolicyUpdateFile**

```c#
 public FileInfo DownloadPolicyUpdateFile(UserInfo user, string contentID)
{
    FileInfo result = null;
    using (TeleEntities context = new TeleEntities())
    {
        DateTime now = GetCurrentDbDataTime();

        var saleContent = context.IDL_SALE_CONTENT
            .Where(x => x.CONTENT_ID == contentID)
            .FirstOrDefault();

        if (saleContent != null)
        {
            if (saleContent.IDL_POLICY_UPDATE_SENT.FirstOrDefault().IDL_POL_UPD_SENT_USED == null)
            {
                IDL_POL_UPD_SENT_USED used = new IDL_POL_UPD_SENT_USED();
                used.POLICY_UPDATE_SENT_ID = saleContent.IDL_POLICY_UPDATE_SENT.FirstOrDefault().POLICY_UPDATE_SENT_ID;
                used.CREATED_ON = now;
                used.CREATED_BY = user.Username;
                context.IDL_POL_UPD_SENT_USED.Add(used);
                context.SaveChanges();

                result = new FileInfo();
                result.Filename = saleContent.FILE_NAME;
                result.Content = saleContent.CONTENT;
                result.ItemCount = saleContent.IDL_POLICY_UPDATE_SENT.FirstOrDefault().IDL_POL_UPDATE_SENT_ITEM.Count();
                return result;
            }
            else
            {
                throw new ApplicationException("ไฟล์ถูก Download ไปแล้ว");
            }
        }
        else
        {
            throw new ApplicationException("ไม่พบไฟล์");
        }
    }
}
```

**DownloadPolicyUpdateFile Method**

``` c#
public FileInfo DownloadPolicyUpdateFile(UserInfo user, string contentID)
{
    ITeleRepository repository = new TeleRepository();
    var result =  repository.DownloadPolicyUpdateFile(user, contentID);

    // send mail
    DateTime now = repository.GetCurrentDateTime();
    var variables = new Dictionary<string, string>() {
        { "Function", "Donwload Policy Update File" },
        { "RunDateTime", now.ToLongDateString() + ' ' + now.ToLongTimeString() },
        { "TotalItem", string.Format("{0:N0}", result.ItemCount) }
    };

    string mailCode = string.Empty;
    //string partyCode = repository.GetPartyCode(user);
    string partyCode = user.Company;
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

    return result;
}
```
### 🎬 **Payment Confirmation** 
🔎 เพื่อส่งข้อมูลยืนยันการได้รับชำระเงินเบี้ยประกันภัย สำหรับรายการชำระผ่านช่องทาง Counter Service (7-11)

<img width="959" alt="Payment Confirimation" src="https://user-images.githubusercontent.com/46476206/147043060-36012a37-7f84-4f84-a192-e9fff09513eb.png">


👨🏻‍💻 **การทำงานของ Code**

🔑 **DownloadPaymentConfirmation**
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
**DownloadPaymentConfirmationFile**
```c#
public FileInfo DownloadPaymentConfirmationFile(UserInfo user, string contentID)
{
    ITeleRepository repository = new TeleRepository();
    var result =  repository.DownloadPaymentConfirmationFile(user, contentID);

    // send mail
    DateTime now = repository.GetCurrentDateTime();
    var variables = new Dictionary<string, string>() {
        { "Function", "Download Payment Confirmation File" },
        { "RunDateTime", now.ToLongDateString() + ' ' + now.ToLongTimeString() },
        { "TotalItem", string.Format("{0:N0}", result.ItemCount) }
    };

    string mailCode = string.Empty;
    //string partyCode = repository.GetPartyCode(user);
    string partyCode = user.Company;
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

    return result;
}
```
**DownloadPaymentConfirmationFile**
```c#
 public FileInfo DownloadPaymentConfirmationFile(UserInfo user, string contentID)
{
    FileInfo result = null;
    using (TeleEntities context = new TeleEntities())
    {
        DateTime now = GetCurrentDbDataTime();

        var saleContent = context.IDL_SALE_CONTENT
            .Where(x => x.CONTENT_ID == contentID)
            .FirstOrDefault();

        if (saleContent != null)
        {
            if(saleContent.IDL1_PAY_CONFIRM_SENT.FirstOrDefault().IDL1_PAY_CONFIRM_SENT_USED == null)
            {
                string payconfirmSentID = saleContent.IDL1_PAY_CONFIRM_SENT.FirstOrDefault().PAY_CONFIRM_SENT_ID;
                IDL1_PAY_CONFIRM_SENT_USED used = new IDL1_PAY_CONFIRM_SENT_USED();
                used.PAY_CONFIRM_SENT_ID = payconfirmSentID;
                used.CREATED_ON = now;
                used.CREATED_BY = user.Username;
                context.IDL1_PAY_CONFIRM_SENT_USED.Add(used);
                context.SaveChanges();

                result = new FileInfo();
                result.Filename = saleContent.FILE_NAME;
                result.Content = saleContent.CONTENT;
                result.ItemCount = saleContent.IDL1_PAY_CONFIRM_SENT.FirstOrDefault().IDL1_PAY_CONFIRM_SENT_ITEM.Count();
                return result;
            }
            else
            {
                throw new ApplicationException("ไฟล์ถูก Download ไปแล้ว");
            }
        }
        else
        {
            throw new ApplicationException("ไม่พบไฟล์");
        }
    }
}
```

### 🎬 **Paycode Follow Up** 
🔎 ติดตามรายการ paycode ที่ยังไม่ได้ชำระเงิน

<img width="959" alt="Paycode Follow up" src="https://user-images.githubusercontent.com/46476206/147043109-21dc5ee2-ce08-4c7c-a3c8-a420f5bd2611.png">


👨🏻‍💻 *การทำงานของ Code*

🔑 **DownloadPaycodeFollowUp**
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

**CreatedNewPaycodeFollowUp**

```c#
public FileInfo CreatedNewPaycodeFollowUp(UserInfo userInfo, PaycodeFollowupParam followUpDate)
{
    FileInfo result = null;

    TRNPPremFollowUp fileDate = null;
    followUpDate.EndDate = followUpDate.EndDate.AddDays(1).AddMilliseconds(-1);
    ITeleRepository party = new TeleRepository();
    //string partyCode = party.GetPartyCode(userInfo);
    string partyCode = userInfo.Company;
    IFollowUpRepository repository = new FollowUpRepository();

    //โหลดรายการที่ต้องติดตามเบี้ย
    var listPremFollowUp = repository.LoadCSOPremFollwUp(followUpDate.BeginDate, followUpDate.EndDate, partyCode);
    if (listPremFollowUp.Count > 0)
    {
        List<string> policyNbrs = listPremFollowUp.Select(s => s.POLICY_NUMBER).Distinct().ToList();
        //หาข้อมูลกรมธรรม์
        List<TRNPCsRnDue> CsoNextDueTxns = repository.GetCsoNextDue(policyNbrs);

        List<TRNPFollowUpLayout> trnpFollowUp = new List<TRNPFollowUpLayout>();
        #region Header
        TRNPFollowUpLayout headerFollowUp = new TRNPFollowUpLayout();
        headerFollowUp.EffectiveDate = "EffectiveDate";
        headerFollowUp.PaidToDate = "PaidToDate";
        headerFollowUp.PaidDate = "PaidDate";
        headerFollowUp.SMSSentDate = "DateSentSMS";
        headerFollowUp.RefNo = "ReferenceNo";
        headerFollowUp.PolicyNumber = "PolicyNo";
        headerFollowUp.Name = "Name";
        headerFollowUp.SurName = "Surname";
        headerFollowUp.Product = "Product";
        headerFollowUp.Plan = "Plan";
        headerFollowUp.Premium = "Premium";
        headerFollowUp.Installments = "Installments";
        headerFollowUp.MobilePhone = "MobilePhone";
        headerFollowUp.PayCode = "PayCode";
        trnpFollowUp.Add(headerFollowUp);
        #endregion

        #region Detail
        List<string> premNoticeIdItemsCancel = new List<string>();

        foreach (var item in listPremFollowUp)
        {
            //ข้อมูลกรมธรรม์ ไว้เช็คกับ Mini ว่าถ้ามีการจ่ายช่องทางอื่น ให้ Cancel รายการ PayItem และไม่นำมารวมใน ReportFollowUp
            var dueTxn = CsoNextDueTxns.Where(w => w.PolicyNumber == item.POLICY_NUMBER).First();

            string miniYL = string.Format("{0:00}", dueTxn.PolicyYear) + string.Format("{0:00}", dueTxn.PolicyLot);
            string followUpYL = string.Format("{0:00}", item.POLICY_YEAR) + string.Format("{0:00}", item.POLICY_LOT);

            //string miniYL = dueTxn.PolicyYear.ToString() + dueTxn.PolicyLot.ToString();
            //string followUpYL = item.POLICY_YEAR.ToString() + item.POLICY_LOT.ToString();

            int miniYearLot = Convert.ToInt32(miniYL);
            int followUpYearLot = Convert.ToInt32(followUpYL);

            if (dueTxn.StatusCode == "10" || dueTxn.StatusCode == "13")
            {
                if (miniYearLot <= followUpYearLot)
                {
                    TRNPFollowUpLayout itemFollowUp = new TRNPFollowUpLayout();
                    itemFollowUp.EffectiveDate = string.Format("{0:yyyyMMdd}", item.EFFECTIVE_DATE);
                    itemFollowUp.PaidToDate = string.Format("{0:yyyyMMdd}", item.PAID_TO_DATE);
                    itemFollowUp.PaidDate = string.Format("{0:yyyyMMdd}", item.PAID_DATE);
                    itemFollowUp.SMSSentDate = string.Format("{0:yyyyMMdd}", item.NOTICE_SMS_ON);
                    itemFollowUp.RefNo = item.REF_NO;
                    itemFollowUp.PolicyNumber = item.POLICY_NUMBER;
                    itemFollowUp.Name = item.INSURED_NAME;
                    itemFollowUp.SurName = item.INSURED_SURNAME;
                    itemFollowUp.Product = item.CAMPAIGN_CODE;
                    itemFollowUp.Plan = item.PRODUCT_PLAN;
                    itemFollowUp.Premium = string.Format("{0:0.00}", item.PREMIUM);
                    itemFollowUp.Installments = string.Format("{0}/{1:00}", item.POLICY_YEAR, item.POLICY_LOT);
                    itemFollowUp.MobilePhone = item.MOBILE_TEL1;
                    itemFollowUp.PayCode = item.CS_PAY_CODE;
                    trnpFollowUp.Add(itemFollowUp);
                }
                else
                {
                    //pay channel offline
                    premNoticeIdItemsCancel.Add(item.PREM_NOTICE_ID);
                }
            }
        }
        #endregion

        //trnpFollowUp.Count = 1 คือมีแค่ header
        if (trnpFollowUp.Count > 1)
        {
            FileHelperEngine engine = new FileHelperEngine(typeof(TRNPFollowUpLayout));
            var fileString = engine.WriteString(trnpFollowUp);
            Encoding utf8 = Encoding.UTF8;

            fileDate = new TRNPPremFollowUp();

            fileDate.Content = utf8.GetBytes(fileString);
            fileDate.FileName = string.Format("{0}{1:yyyyMMdd}.txt", "CSOFOLLOWUP_", DateTime.Today);
            fileDate.ItemCount = trnpFollowUp.Count() - 1;

            //insert cancel 
            repository.CancelPayChannelOffline(premNoticeIdItemsCancel, "Offline Payment");

            // Add Content
            var premFollowUp = repository.AddNewPremFollowUpContent(fileDate.Content, fileDate.FileName, 0, userInfo.Username);
            result = new FileInfo();
            result.Content = fileDate.Content;
            result.Filename = fileDate.FileName;

            // Send Mail
            SendMail(partyCode, fileDate.ItemCount, "Download CounterService FollowUp File");
        }

    }

    return result;
}
```


### 🎬 **Paycode Reply** 
🔎 รายการที่ทำการตอบกลับไปทาง I-Direct เพื่อให้ทาง I-Direct Download ไปติดตามลูกค้า

<img width="956" alt="Paycode Reply" src="https://user-images.githubusercontent.com/46476206/147043128-09f989bf-e689-46b7-b354-1df02850edfa.png">

👨🏻‍💻 **การทำงานของ Code**

🔑 **DownloadPaycodeReply**
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

**DownloadPaycodeReplyFile**

```c#
public FileInfo DownloadPaycodeReplyFile(UserInfo user, string contentID)
{
    ITeleRepository repository = new TeleRepository();
    var result =  repository.DownloadPaycodeReplyFile(user, contentID);

    // send mail
    DateTime now = repository.GetCurrentDateTime();
    var variables = new Dictionary<string, string>() {
        { "Function", "Download Paycode Reply File" },
        { "RunDateTime", now.ToLongDateString() + ' ' + now.ToLongTimeString() },
        { "TotalItem", string.Format("{0:N0}", result.ItemCount) }
    };

    string mailCode = string.Empty;
    //string partyCode = repository.GetPartyCode(user);
    string partyCode = user.Company;
    if (partyCode == PartyCode.IDBL)
    {
        mailCode = MailCode.TELE_IDBL_FOLLOWUP;
    }
    else if (partyCode == PartyCode.TVD)
    {
        mailCode = MailCode.TELE_TVD_FOLLOWUP;
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

    return result;
}
```

**DownloadPaycodeReplyFile**

```c# 
public FileInfo DownloadPaycodeReplyFile(UserInfo user, string contentID)
{
    FileInfo result = null;
    using (TeleEntities context = new TeleEntities())
    {
        DateTime now = GetCurrentDbDataTime();

        var saleContent = context.IDL_SALE_CONTENT
            .Where(x => x.CONTENT_ID == contentID)
            .FirstOrDefault();

        if (saleContent != null)
        {
            if (saleContent.CSO4_PAY_CODE_REPLY.FirstOrDefault().CSO4_PAY_CODE_REPLY_USED == null)
            {
                string paycodeReplyID = saleContent.CSO4_PAY_CODE_REPLY.FirstOrDefault().PAY_CODE_REPLY_ID;
                CSO4_PAY_CODE_REPLY_USED used = new CSO4_PAY_CODE_REPLY_USED();
                used.PAY_CODE_REPLY_ID = paycodeReplyID;
                used.CREATED_ON = now;
                used.CREATED_BY = user.Username;
                context.CSO4_PAY_CODE_REPLY_USED.Add(used);
                context.SaveChanges();

                result = new FileInfo();
                result.Filename = saleContent.FILE_NAME;
                result.Content = saleContent.CONTENT;
                result.ItemCount = saleContent.CSO4_PAY_CODE_REPLY.FirstOrDefault().CSO4_PAY_CODE_REPLY_ITEM.Count();
                return result;
            }
            else
            { 
                throw new ApplicationException("ไฟล์ถูก Download ไปแล้ว");
            }
        }
        else
        {
            throw new ApplicationException("ไม่พบไฟล์");
        }
    }
}
```
### 🎬 **Recurring Follow Up** 
🔎 ติดตามรายการตัดบัตรที่ยังไม่ได้ชำระเงิน

<img width="958" alt="Recurring Follow UP" src="https://user-images.githubusercontent.com/46476206/147043176-3b9bd2ca-8762-4cac-8775-1160a03fe604.png">

👨🏻‍💻 **การทำงานของ Code**

🔑 **DownloadRecurringFollowUp**
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

👨🏻‍💻 **การทำงานของ Code**

🔑 **DownloadSaleLead**
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

**DownloadSaleLeadFile**

```c#
 public FileInfo DownloadSaleLeadFile(UserInfo user, string contentID)
{
    ITeleRepository repository = new TeleRepository();
    var result = repository.DownloadSaleLeadFile(user, contentID);
    // send mail
    DateTime now = repository.GetCurrentDateTime();
    var variables = new Dictionary<string, string>() {
        { "Function", "Donwload Sale Lead File" },
        { "RunDateTime", now.ToLongDateString() + ' ' + now.ToLongTimeString() },
        { "TotalItem", string.Format("{0:N0}", result.ItemCount) }
    };

    EMailFromDb email = new EMailFromDb();
    email.Send(MailCode.TELE_IDBL_FOLLOWUP, variables);

    return result;
}
```

