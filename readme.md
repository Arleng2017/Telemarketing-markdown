#  Telemarketing
Gitlab Repo 
> http://gitlab.bla.co.th/telemarketing/telemarketing.api
        
💻 หน้า UI หลัก 

<img width="949" alt="main desktop" src="https://user-images.githubusercontent.com/46476206/147031685-0121d171-49b2-4d56-a437-488bb3786d69.png">

😀 หลักการทำงานของโค้ด 😀 

## Upload

### First Billing
รายการส่งตัดบัตรเครดิตงวดแรก

``` c#

 public void UploadYesSale(ObjectParam param)
        {
            DelimitedFileEngine engine = new DelimitedFileEngine(typeof(YesSaleLayout));
            ITeleRepository repository = new TeleRepository();
            IEnumerable<YesSaleLayout> items = null;

            string strContentFile = Encoding.GetEncoding(874).GetString(param.File.Content);
            try
            {
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
            repository.UploadYesSale(param, items);
            
            // send mail
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
###  Yes Sale
เพื่อจัดส่งรายการ Yes sale ที่ตรวจสอบคุณภาพเรียบร้อยแล้วให้ BLA พิจารณาออกกรมธรรม์

``` c#

```
``` c#

```
``` c#

```
``` c#

```
``` c#

```
###  Cancel Case
เพื่อจัดส่งรายการลูกค้าที่ได้รับการติดต่อจาก Confirmation call แต่มีความประสงค์จะยกเลิกกธ. ให้ BLA

```

```
###  Application Info
เพื่อส่งข้อมูลคำขอเอาประกันภัย สำหรับรายการชำระผ่านช่องทาง Counter Service (7-11)
```

```

## Download
###  First Billing
ผลการตัดบัตรเครดิตงวดแรก

```

```
###  Policy Update
เพื่อ BLA จัดส่งรายการลูกค้าที่ออกกรมธรรม์แล้ว ให้ IDB โดยระบุ Policy no. Mailing date เพื่อใช้ติดต่อทำ Confirmation call 
การส่งข้อมูล จัดส่งเฉพาะรายการที่ส่งกธ.ให้ลูกค้าแล้ว ตาม Transaction date"

```

```
###  Policy Cancel
เพื่อ BLA จัดส่งรายการยกเลิกกธ. ให้ IDB update Policy status

```

```
###  Payment Confirmation
เพื่อส่งข้อมูลยืนยันการได้รับชำระเงินเบี้ยประกันภัย สำหรับรายการชำระผ่านช่องทาง Counter Service (7-11)

```

```
###  Paycode Follow Up
ติดตามรายการ paycode ที่ยังไม่ได้ชำระเงิน

```

```
###  Paycode Reply
รายการที่ทำการตอบกลับไปทาง I-Direct เพื่อให้ทาง I-Direct Download ไปติดตามลูกค้า

```

```
###  Recurring Follow Up
ติดตามรายการตัดบัตรที่ยังไม่ได้ชำระเงิน

```

```
###  Sale Lead
เพื่อ BLA จัดส่งรายการ DRTV's Call List

```

```

