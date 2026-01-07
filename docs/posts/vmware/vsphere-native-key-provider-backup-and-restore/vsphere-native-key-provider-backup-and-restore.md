---
title: vSphere Native Key Provider (NKP) 設定、備份與還原
description: 紀錄如何在 vCenter 中建立 Native Key Provider (NKP)，並詳細說明備份與還原操作流程，確保加密機制順利運作並建立完善的災難還原計畫。
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
- [2. 運作需求](#2-運作需求)
- [3. 新增 Native Key Provider](#3-新增-native-key-provider)
- [4. 備份 NKP](#4-備份-nkp)
  - [4.1. 透過 vSphere Client 備份](#41-透過-vsphere-client-備份)
  - [4.2. 透過 CLI 方式備份](#42-透過-cli-方式備份)
- [5. 還原 NKP](#5-還原-nkp)

<div class="page-break"/>

## 1. 環境說明

- **vCenter 版本**：8.0 U3

## 2. 運作需求

- vCenter Server 與 ESXi 主機必須運行 **vSphere 7.0 Update 2** 或更新版本。​
- ESXi 主機必須已納入**叢集 (Cluster)** 管理。​

<div class="page-break"/>

## 3. 新增 Native Key Provider

請在 vCenter 的「金鑰提供者」設定頁面中點選新增：

![Add Native Key Provider](images/img-1.png)

![Add Native Key Provider](images/img-2.png)

| 項次 | 參數名稱 | 功能描述 |
| :--: | :--- | :--- |
|  1   | 名稱 | 此 NKP 的唯一識別名稱，可自定義。 |
|  2   | 僅對受 TPM 保護的 ESXi 主機使用金鑰提供者 | **勾選**：僅限具備 TPM 晶片的主機使用。<br>**取消勾選**：無硬體 TPM 限制，相容性較高。 |

![Add Native Key Provider](images/img-3.png)

!!! info "關鍵步驟"
    新增完成後，NKP 的狀態預設為「未備份」。**必須先執行一次備份**，讓系統狀態標示為「已備份」，之後才能正式啟用主機加密或虛擬機加密相關功能。

<div class="page-break"/>

## 4. 備份 NKP

!!! warning "UI 備份失敗的常見原因"
    在標準操作下，點擊「備份」即可下載金鑰。但若 vCenter 的 **PNID (主機名稱)** 未正確配置（例如顯示為 `localhost`），UI 生成的下載 URL 會因為位址無效而失敗。
    
    若您看到「備份原生金鑰提供者失敗」訊息，請改用 **4.2 節** 的 CLI 備份方式。

    ![Backup Failed](images/img-10.png)

<div class="page-break"/>

### 4.1. 透過 vSphere Client 備份

這是最推薦的標準操作流程：

![Backup Native Key Provider](images/img-8.png)

![Backup Native Key Provider](images/img-9.png)

| 項次 | 參數名稱 | 描述 |
| :--: | :--- | :--- |
|  1   | 使用密碼保護原生金鑰提供者資料 (建議) | 勾選此項後，備份檔將會受到密碼保護。 |
|  2   | 密碼/驗證密碼 | 設定還原時所需的密碼，**請務必妥善保管**。 |
|  3   | 我已將密碼儲存在安全的位置 | 勾選此確認項後方可點擊備份。 |

<div class="page-break"/>

### 4.2. 透過 CLI 方式備份

**步驟一：進入 Shell 模式**
以 SSH 方式登入 vCenter 後進入 shell：

```bash
Command> {==shell==}
Shell access is granted to root
root@localhost [ ~ ]# 
```

**步驟二：取得下載 NKP Token**
請將 `<nkp名稱>` 替換為實際名稱。系統會要求輸入 vCenter SSO 管理員帳密。

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

**步驟三：下載備份檔**
在可連通 vCenter 的電腦使用 `curl` 下載。**注意：** 須將 URL 中的 `localhost` 改為 vCenter 的實際 IP 或 FQDN。

```bash
# 指令格式
curl -k \
  -H "Authorization: Bearer <步驟二獲得的 token 字串>" \
  "https://<vCenter-IP-or-FQDN>/cryptomanager/kms/<NKP名稱>" \
  -o <自訂名稱>.p12

# 實務範例
curl -k \
  -H "Authorization: Bearer esJhbvciOiJIUzI1NiIsInR5cdI6IkpXVCJ9..." \
  '[https://192.168.1.11/cryptomanager/kms/nkp1](https://192.168.1.11/cryptomanager/kms/nkp1)' \
  -o nkp1.p12
```

<div class="page-break"/>

## 5. 還原 NKP

當進行環境遷移或災後復原時，請依下列步驟操作：

![Restore Native Key Provider](images/img-4.png)

![Restore Native Key Provider](images/img-5.png)

上傳先前備份的 `.p12` 檔案，並輸入當時設定的保護密碼。

![Restore Native Key Provider](images/img-6.png)

根據硬體條件決定是否啟用 TPM 保護限制。

![Restore Native Key Provider](images/img-7.png)

**後續檢查：**

1. 確保下方的檢查清單皆呈現**綠色勾勾**。
2. **重要：** 若環境中僅有此 NKP，必須將其**設為預設值**。否則虛擬機器的加密選項可能會顯示為反灰（不可選），導致功能無法啟用。

<div class="page-break"/>

<h2 class="no-print">參考資料</h2>

- [Configure a vSphere Native Key Provider](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/7-0/vsphere-security-7-0/configuring-and-managing-vsphere-native-key-provider/configure-a-vsphere-native-key-provider.html){:target="_blank" class="no-print"}
- [Back up a vSphere Native Key Provider](https://techdocs.broadcom.com/tw/zh-tw/vmware-cis/vsphere/vsphere/7-0/vsphere-security-7-0/configuring-and-managing-vsphere-native-key-provider/back-up-a-vsphere-native-key-provider.html){:target="_blank" class="no-print"}
- [Restore a vSphere Native Key Provider Using the vSphere Client](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/7-0/vsphere-security-7-0/configuring-and-managing-vsphere-native-key-provider/recovering-a-vsphere-native-key-provider/restore-a-vsphere-native-key-provider-using-the-vsphere-client.html){:target="_blank" class="no-print"}