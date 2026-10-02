# Logic Apps — Outlook email → attachments

ตัวอย่าง workflow สำหรับ Azure Logic Apps ที่รับอีเมลจาก Outlook แล้วดึงไฟล์แนบไปเก็บที่
Azure Blob Storage หรือ SharePoint พร้อมเอกสารเทียบ trigger V1 vs V2 แบบ field-by-field

> โฟลเดอร์นี้ไม่เกี่ยวข้องกับแอป `payment/` ใน repo นี้ เป็นไฟล์ประกอบการทำ integration แยกต่างหาก

## ไฟล์ในโฟลเดอร์นี้

| ไฟล์ | Connector | Trigger | ปลายทาง |
| --- | --- | --- | --- |
| `workflows/outlook-attachments-to-blob/workflow.json` | Office 365 Outlook (`office365`) | `When a new email arrives (V3)` | Azure Blob |
| `workflows/outlookcom-attachments-to-blob/workflow.json` | Outlook.com (`outlook`) — **บัญชีส่วนตัวเท่านั้น** | `When a new email arrives (V2)` | Azure Blob |
| `workflows/outlook-mention-attachments-to-sharepoint/workflow.json` | Office 365 Outlook (`office365`) | `When a new email mentioning me arrives (V2)` | SharePoint |
| `consumption/outlook-attachments-to-blob.definition.json` | Office 365 Outlook | เหมือนไฟล์แรก แต่เป็นรูปแบบ Consumption | Azure Blob |
| `connections.json` | — | ตัวอย่าง managed API connections สำหรับ Logic Apps Standard | — |

## เลือก connector ให้ถูกตัวก่อน (เรื่องนี้ต้องทำก่อนอย่างอื่น)

Logic Apps มี Outlook connector **สองตัวที่คนละอันกันโดยสิ้นเชิง** และไอคอนหน้าตาคล้ายกันมาก

| Connector | Connection key | รับบัญชีแบบไหน |
| --- | --- | --- |
| **Office 365 Outlook** | `office365` | Work or school account (Microsoft Entra ID) เช่น `someone@company.co.th` |
| **Outlook.com** | `outlook` | Personal Microsoft account เท่านั้น เช่น `@outlook.com`, `@hotmail.com`, `@live.com` |

ทั้งสองตัวไม่สามารถใช้บัญชีข้ามกันได้ เพราะคุยกับ identity system คนละระบบ

### อาการเมื่อเลือกผิด

ถ้าหน้า Create connection ขึ้นข้อความว่า

> Sign in to create a connection to **Outlook.com**

แปลว่า trigger/action ที่วางไว้เป็นตัวของ connector **Outlook.com** ถ้าเอาบัญชีบริษัทไป sign in
ที่หน้านี้ จะได้ผลลัพธ์เป็น

> We couldn't find a Microsoft account. Try entering your details again, or create an account.

ข้อความนี้ไม่ได้แปลว่าบัญชีมีปัญหา แต่แปลว่า Microsoft ไปค้นบัญชีในฐานข้อมูลของ MSA
(Personal account) ซึ่งไม่มีบัญชีขององค์กรอยู่ในนั้นตั้งแต่แรก

### วิธีแก้

สิ่งที่**ไม่ช่วย**ในเคสนี้ คือ เปิด incognito, ล้าง session, กด Change connection แล้ว sign in ใหม่
หรือตรวจ API connections ใน Azure Portal เพราะปัญหาอยู่ที่ตัว connector ที่เลือกไว้ ไม่ใช่ session
หรือ credential

ต้องเปลี่ยน connector แทน

1. ลบ trigger ตัวเดิมออกจาก designer
2. Add a trigger แล้วค้นด้วยคำว่า `Office 365 Outlook` (ไม่ใช่ `Outlook.com`)
3. เลือก `When a new email arrives (V3)` หรือ `When a new email mentioning me arrives (V2)`
4. Sign in ด้วยบัญชีบริษัท หน้า sign in ที่ถูกต้องจะพาไปที่ login ของ tenant องค์กร

ถ้าเคยลองสร้าง connection ค้างไว้ ให้ไปลบ connection เสียของเก่าทิ้งที่
Azure Portal → Logic App → API connections ด้วย ไม่งั้นจะมี connection ที่ status เป็น Error ค้างอยู่

วิธีเช็กในไฟล์ `workflow.json` ว่าตอนนี้อยู่ connector ตัวไหน ให้ดูที่ `referenceName` หรือ `path`

```json
"connection": { "referenceName": "outlook" }      // Outlook.com  — บัญชีส่วนตัว
"connection": { "referenceName": "office365" }    // Office 365 Outlook — บัญชีบริษัท
```

สำหรับบัญชีอย่าง `@banpu.co.th` ให้ใช้ไฟล์ `workflows/outlook-attachments-to-blob/` หรือ
`workflows/outlook-mention-attachments-to-sharepoint/` ส่วน `workflows/outlookcom-attachments-to-blob/`
มีไว้เทียบเฉยๆ ใช้กับบัญชีองค์กรไม่ได้

### ถ้าเลือก Office 365 Outlook แล้วยัง sign in ไม่ผ่าน

กรณีนั้นค่อยไปไล่ตามที่คุณว่าไว้ คือเช็ก tenant และ session

- เข้า [myaccount.microsoft.com](https://myaccount.microsoft.com) ด้วยบัญชีเดียวกันเพื่อยืนยันว่าบัญชีใช้งานได้
- ถ้ามีหลาย tenant ให้ดูว่า Logic App อยู่ tenant เดียวกับบัญชีอีเมลหรือเปล่า ถ้าคนละ tenant
  ต้องมี guest access หรือย้าย Logic App
- บาง tenant ปิด third-party app consent ไว้ ต้องให้ Entra ID admin อนุมัติ connector ก่อน
- ค่อยลอง incognito เป็นขั้นสุดท้าย เผื่อมี session ซ้อนกัน

## Standard vs Consumption — จุดที่โค้ดต่างกัน

สองแบบนี้ไม่สามารถ copy ข้ามกันได้ตรง ๆ เพราะอ้าง connection คนละวิธี

Logic Apps **Standard** (`workflow.json` + `connections.json`) อ้างด้วย `referenceName`

```json
"host": { "connection": { "referenceName": "office365" } }
```

Logic Apps **Consumption** (ARM / code view) อ้างผ่าน parameter `$connections`

```json
"host": { "connection": { "name": "@parameters('$connections')['office365']['connectionId']" } }
```

ส่วน **Power Automate** มีแค่ Peek code ซึ่งอ่านได้อย่างเดียว แก้ไม่ได้ ต้องประกอบ action จาก designer

## แนวทางที่แนะนำ: ไม่ต้องเรียก Get attachment

ถ้าตั้ง `includeAttachments: true` ที่ trigger ตัว trigger จะส่ง `contentBytes` (base64 ของไฟล์จริง)
มาใน `triggerBody()?['attachments']` ให้เลย แปลว่าวนลูปแล้วเขียนไฟล์ได้ทันที ไม่ต้องยิง
`Get attachment (V2)` เพิ่มอีกรอบ ซึ่งช่วยลดจำนวน action call และลดโอกาสโดน throttle

```json
"body": "@base64ToBinary(items('For_each_attachment')?['contentBytes'])"
```

`base64ToBinary` จำเป็น เพราะ connector ส่ง base64 string มา ถ้าส่ง string ดิบเข้าไปตรง ๆ
ไฟล์ปลายทางจะกลายเป็นไฟล์ข้อความที่มีเนื้อหาเป็น base64 แทนที่จะเป็นไฟล์จริง

ควรเรียก `Get attachment (V2)` เมื่อ

- ได้ message id มาจากทางอื่น เช่นจาก trigger หรือ action ที่ไม่ส่ง `contentBytes` มาด้วย
- ต้องการ metadata ของไฟล์เพิ่มเติมที่ trigger ไม่ได้ให้
- ต้องการแยกการดึงไฟล์ออกมาเป็น action ของตัวเอง เพื่อใส่ retry policy หรือ error handling เฉพาะจุด

ไฟล์ `outlook-mention-attachments-to-sharepoint/workflow.json` ทำแบบนี้ไว้ให้ดูเป็นตัวอย่าง คือวน
`attachments` จาก trigger เพื่อเอา `id` แล้วค่อยเรียก `Get attachment (V2)` ทีละไฟล์
ถ้าไม่ต้องการ action เพิ่ม ก็ใช้ `items('For_each_attachment')?['contentBytes']` จาก trigger ได้ตรง ๆ เลย

## V1 vs V2 — field-by-field

trigger ทั้ง `OnNewEmail` และ `OnNewMentionMeEmail` ใช้ schema ชุดเดียวกัน
จุดเปลี่ยนใหญ่สุดจาก V1 → V2 คือ V2 เปลี่ยนไปใช้ชื่อ field แบบ Microsoft Graph (camelCase)

### Field ของอีเมล

| V1 (PascalCase) | V2 / V3 (camelCase) | หมายเหตุ |
| --- | --- | --- |
| `Id` | `id` | id ของ message ฝั่ง Graph ใช้ตัวนี้ส่งต่อให้ `Get attachment` |
| `MessageId` | `internetMessageId` | คนละตัวกับ `id` เป็น RFC822 Message-ID |
| `From` | `from` | string อีเมลเดียว |
| `To` / `Cc` / `Bcc` | `toRecipients` / `ccRecipients` / `bccRecipients` | ยังเป็น string คั่นด้วย `;` ไม่ใช่ array |
| `Subject` | `subject` | |
| `Body` | `body` | |
| — | `bodyPreview` | เพิ่มใน V2 เป็น plain text ตัดสั้น |
| `Importance` | `importance` | |
| `HasAttachment` | `hasAttachments` | **เติม s** ตรงนี้ทำ expression พังบ่อยที่สุดตอน migrate |
| `DateTimeReceived` | `receivedDateTime` | |
| `IsRead` | `isRead` | |
| `IsHtml` | `isHtml` | |
| `ConversationId` | `conversationId` | |
| — | `webLink` | เพิ่มใน V2 เปิดอีเมลใน OWA ได้ตรง ๆ |
| `Attachments` | `attachments` | |

### Field ของ attachment แต่ละตัว

| V1 | V2 / V3 | หมายเหตุ |
| --- | --- | --- |
| `Id` | `id` | |
| `Name` | `name` | |
| `ContentBytes` | `contentBytes` | base64 ต้องผ่าน `base64ToBinary` ก่อนเขียนไฟล์ |
| `ContentType` | `contentType` | |
| `Size` | `size` | |
| — | `isInline` | เพิ่มใน V2 ใช้กรองรูปลายเซ็น/รูปใน body ออก |

`isInline` คือเหตุผลที่ workflow ตัวอย่างมี condition `Skip_inline_images` เพราะถ้าไม่กรอง
อีเมลที่มีโลโก้ในลายเซ็นจะถูกนับเป็นไฟล์แนบและถูกเซฟลง storage ทุกฉบับ

### Query parameter ของ trigger

| Parameter | V1 | V2 | V3 |
| --- | --- | --- | --- |
| `folderPath` | มี | มี | มี (เลือกจาก picker) |
| `fetchOnlyUnread` | มี | มี | มี |
| `fetchOnlyWithAttachment` | มี | มี | มี |
| `includeAttachments` | มี | มี | มี |
| `subjectFilter` | มี | มี | มี |
| `importance` | ไม่มี | มี | มี |
| `from` / `to` / `cc` / `toOrCc` | ไม่มี | บางส่วน | มีครบ |

`When a new email mentioning me arrives` มีถึงแค่ V2 (`OnNewMentionMeEmailV2`) ส่วน
`When a new email arrives` ของ Office 365 Outlook ไปถึง V3 แล้ว แต่ Outlook.com (ตาม connector
ในสกรีนช็อต) ยังอยู่ที่ V2 ซึ่งเป็นเหตุผลที่ไฟล์ `outlookcom-*` ใช้ path `/v2/Mail/OnNewEmail`

## ข้อควรระวัง

**`splitOn` กับ `For each` คนละชั้นกัน** — `"splitOn": "@triggerBody()?['value']"` แตก *อีเมลหลายฉบับ*
ออกเป็นคนละ run ส่วน `For each` วน *ไฟล์แนบหลายไฟล์* ในอีเมลฉบับเดียว ต้องมีทั้งคู่

**`concurrency.repetitions: 1`** — บังคับให้ loop ทำทีละไฟล์ ถ้าปล่อย default (20 ขนาน)
จะโดน throttle ของ Outlook/SharePoint ง่ายมากเวลาอีเมลมีไฟล์แนบเยอะ

**ชื่อไฟล์ซ้ำ** — ตัวอย่างเติม `utcNow('yyyyMMddHHmmssfff')` นำหน้าชื่อไฟล์ ถ้าไม่ทำ ไฟล์ชื่อซ้ำ
เช่น `invoice.pdf` จะทับกันหรือทำให้ action fail แล้วแต่ connector ปลายทาง

**ไฟล์ใหญ่** — `transferMode: Chunked` จำเป็นสำหรับไฟล์เกิน ~50 MB แต่ connector ฝั่ง Outlook
เองก็มีเพดานขนาด attachment ของตัวเอง ไฟล์ที่ใหญ่กว่านั้นต้องไปทาง Graph API โดยตรง

**trigger payload limit** — ถ้าเปิด `includeAttachments: true` แล้วอีเมลมีไฟล์แนบรวมขนาดใหญ่
trigger อาจ fail ไปเลย กรณีนี้ให้ปิด `includeAttachments` แล้วใช้รูปแบบ list + `Get attachment` แทน

**`id` ของ message มีอักขระพิเศษ** — ต้องหุ้ม `encodeURIComponent()` เสมอเวลาเอาไปต่อใน path
ของ `Get attachment` ตัวอย่างในไฟล์ทำไว้แล้ว

## การนำไปใช้

ค่าที่ต้องแก้ก่อนใช้งานจริง

- `AccountNameFromSettings` ใน path ของ Blob → เปลี่ยนเป็นชื่อ storage account หรือปล่อยไว้ถ้า
  ตั้ง app setting ของ connection ไว้แล้ว
- `sharePointSiteUrl` และ `sharePointFolderPath` ใน workflow ของ SharePoint
- `blobFolderPath` / `mailFolderPath` ตามโครงสร้างจริง

ขั้นตอนที่เร็วที่สุดคือสร้าง connection จาก designer ก่อนหนึ่งรอบ (ตามหน้าจอ Create connection →
Sign in) แล้วค่อยเปิด Code view เอา definition จากไฟล์นี้ไปวางทับ เพราะ `connectionId` และ
`connectionRuntimeUrl` จะถูกเติมให้อัตโนมัติ

path ของ `Get attachment (V2)` ในไฟล์ SharePoint เขียนตามรูปแบบ `/v2/Mail/{messageId}/Attachments/{attachmentId}`
ซึ่งเป็นรูปแบบของ connector รุ่นปัจจุบัน แต่ถ้า connector ในเทนแนนต์ของคุณต่างออกไป ให้ลากตัว action
ลงใน designer หนึ่งครั้งแล้วเทียบ path จาก Code view ก่อนนำไป deploy

## เทียบกับ Microsoft Graph โดยตรง

ถ้าจะทำผ่าน Graph (เช่นใน n8n หรือโค้ดเอง) แทน connector

```http
GET /me/messages/{messageId}/attachments
GET /me/messages/{messageId}/attachments/{attachmentId}/$value
```

endpoint แรกคืนรายการพร้อม `contentBytes` (base64) เทียบเท่ากับที่ connector V2 ส่งมา
ส่วน `/$value` คืน raw bytes ตรง ๆ ไม่ต้อง decode ซึ่งเหมาะกับไฟล์ใหญ่มากกว่า
scope ที่ต้องใช้คือ `Mail.Read`
