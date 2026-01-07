---
title: vSphere Native Key Provider (NKP) 設定、備份與還原
description: 紀錄如何在 vCenter 中建立 Native Key Provider（NKP），並詳細說明 NKP 備份與還原操作，確保加密機制順利運作及災難還原。
date: 2025-12-02
categories:
    - VMware
tags:
    - vCenter
    - Native Key Provider
comments: true
banner: img.png
draft: false
---

<h2>目錄</h2>

- [1. 環境說明](#1-環境說明)
- [2. 需求](#2-需求)
- [3. 新增](#3-新增)
- [4. 備份](#4-備份)
  - [4.1. 透過 vSphere Client 備份](#41-透過-vsphere-client-備份)
  - [4.2. 透過  CLI  方式備份](#42-透過--cli--方式備份)
- [5. 還原NKP](#5-還原nkp)

<div class="page-break"/>

## 1. 環境說明

- vCenter版本：8.0 U3

## 2. 需求

- vCenter Server 與 ESXi 主機必須運行 vSphere 7.0 Update 2 或更新版本。​
- 必須將 ESXi 主機設定成叢集（Cluster）。​

<div class="page-break"/>

## 3. 新增

![Add Native Key Provider](images/img-1.png)

![Add Native Key Provider](images/img-2.png)

| 項次 |                   參數                    | 描述                                                               |
| :--: | :---------------------------------------: | ------------------------------------------------------------------ |
|  1   |                   名稱                    | 此NKP名稱，可自定義                                                |
|  2   | 僅對受 TPM 保護的 ESXi 主機使用金鑰提供者 | 若勾選則此NKP就只能在已經啟用TPM的ESXI主機使用，取消勾選則無此限制 |

![Add Native Key Provider](images/img-3.png)

在完成新增 NKP 後，必須先完成一次備份並於 vCenter 中標示為「已備份」，之後才能在叢集上啟用主機加密模式與相關加密功能。

<div class="page-break"/>

## 4. 備份

!!! warning
    在預設情況下，  vSphere  提供在  vSphere Client  介面中直接按下「備份」按鈕即可
    下載  NKP  。當  vCenter  的主機名稱 （Hostname / PNID） 沒有正確設定，而是顯示為
      localhost  或未設定時，備份下載連結會內嵌這個錯誤的主機名稱，導致瀏覽器產生錯誤的 URL
     ，最終出現「備份原生金鑰提供者失敗」訊息，無法透過 UI 取得備份檔，針對此情形就只能使用CLI
     方式備份
     ![Backup Failed](images/img-10.png)

<div class="page-break"/>

### 4.1. 透過 vSphere Client 備份

![Backup Native Key Provider](images/img-8.png)

![Backup Native Key Provider](images/img-9.png)

| 項次 |                 參數                  | 描述                                           |
| :--: | :-----------------------------------: | ---------------------------------------------- |
|  1   | 使用密碼保護原生金鑰提供者資料 (建議) | 勾選此選項才會使用密碼保護金鑰                 |
|  2   |             密碼/驗證密碼             | 輸入要設定的密碼，後續若要還原時需要輸入此密碼 |
|  3   |      我已將密碼儲存在安全的位置       | 需勾選才可備份                                 |

<div class="page-break"/>

### 4.2. 透過  CLI  方式備份

**進入 shell 模式**

以  SSH  方式登入到 vCenter 然後進入 shell 模式

```bash
Command> {==shell==}
Shell access is granted to root
root@localhost [ ~ ]# 
```

**取得下載 NKP Token**

此處的  nkp  名稱要填入新增  nkp  時的名稱，提示輸入帳號密碼時需要輸入  vCenter SSO Admin  帳密，輸入完後提示是否儲存帳密資訊時輸入   `n`

```bash
# 指令格式

dcli com vmware vcenter cryptomanager kms providers export --provider <nkp名稱>
```

```bash
# 範例

root@localhost [ ~ ]# {==dcli com vmware vcenter cryptomanager kms providers export --provider nkp1==}
Username: {==administrator@vsphere.local==}
Password: {==*********==}
Do you want to save credentials in the credstore? (y or n) [y]:{==n==}
location:
   download_token:
      expiry: 2025-12-02T15:15:29.000Z
      token: esJhbvciOiJIUzI1NiIsInR5cdI6IkpXVCJ9...
   url: https://localhost/cryptomanager/kms/nkp1
type: LOCATION
```

<div class="page-break"/>

**下載NKP**

在要下載NKP的電腦使用curl指令下載，token需填入上一步驟顯示的token，網址部分為上一步驟顯示的url但需要將localhost改成實際的vCenter IP

```bash
# 指令格式

curl -k \
  -H "Authorization: Bearer <download_token 中的 token 字串>" \
  "https://<vCenter-IP-or-FQDN>/cryptomanager/kms/<NKP_name>" \
  -o <NKP_name>.p12
```

```bash
# 範例

curl -k \
  -H "Authorization: Bearer esJhbvciOiJIUzI1NiIsInR5cdI6IkpXVCJ9..." \
  'https://192.168.1.11/cryptomanager/kms/nkp1' \
  -o nkp1.p12
```

<div class="page-break"/>

## 5. 還原NKP

![Restore Native Key Provider](images/img-4.png)

![Restore Native Key Provider](images/img-5.png)

選擇備份的  NKP  檔案，若備份時有設定密碼則需要在此處輸入密碼

![Restore Native Key Provider](images/img-6.png)

若勾選此項目則此  NKP  就只能在已經啟用  TPM  的  ESXI  主機使用，取消勾選則無此限制

![Restore Native Key Provider](images/img-7.png)

匯入完成後下方三個項目需都是三個綠色勾勾，若此  vCenter  沒有其他NKP的話需要將匯入的  NKP  設為預設值，否則 vSphere Client 中的虛擬機器加密選項可能會呈現反灰狀態，無法啟用或調整加密設定。

<div class="page-break"/>

<h2 class="no-print">參考資料</h2>

- [Configure a vSphere Native Key Provider](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/7-0/vsphere-security-7-0/configuring-and-managing-vsphere-native-key-provider/configure-a-vsphere-native-key-provider.html){:target="_blank" class="no-print"}
- [Back up a vSphere Native Key Provider](https://techdocs.broadcom.com/tw/zh-tw/vmware-cis/vsphere/vsphere/7-0/vsphere-security-7-0/configuring-and-managing-vsphere-native-key-provider/back-up-a-vsphere-native-key-provider.html){:target="_blank" class="no-print"}
- [Restore a vSphere Native Key Provider Using the vSphere Client](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/7-0/vsphere-security-7-0/configuring-and-managing-vsphere-native-key-provider/recovering-a-vsphere-native-key-provider/restore-a-vsphere-native-key-provider-using-the-vsphere-client.html){:target="_blank" class="no-print"}