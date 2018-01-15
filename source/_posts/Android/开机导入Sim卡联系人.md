---
title: 开机导入Sim卡联系人
permalink: import-Sim-card-contacts-when-boot
categories:
  - Android
tags:
  - android
  - 面试
  - contacts
date: 2018-01-15 22:54:34
---

## ContactsProvider
１.ContactsProvider.apk会启动一个广播接收器SystemStartReceiver来接收BOOT_COMPLETED的广播。

２.ContactsProvider收到该广播后，将raw_contacts表里所有非本地联系人的数据都删除掉。
然后读取当前是否存在Sim卡，若存在，则发SYNC_ICC_CARD_CONTACTS广播。

3.SystemStartReceiver收到广播后启动一个thread来执行读取sim卡联系人，并发送MSG_NOTIFY_ICC_LOADING的message给主线程，主线程收到后发loadicccontacts广播。

4.开始读取ICC数据库的内容，IccProvider中会执行loadFromEf()，获得IIccPhoneBook接口，通过AIDL调用getAdnRecordsInEf()方法，获取sim卡上的全部联系人数据，返回cursor并装载成ArrayList。

5.子线程给主线程发送MSG_NOTIFY_ICC_CHECKFINISHING的message，表示ICC读取完毕。主线程会发loadicccontacts广播。

6.子线程发送消息MSG_INSERT_NEW_CONTACTS给主线程，进行数据库批量处理的操作，即把几个ContentProviderOperation打包在一起，Transaction事务机制，通过applyBatch()方法，主线程将这些sim卡联系人逐个的添加到raw_contacts表和data表中.

## Contacts
Contacts 收到loadicccontacts的广播，设置当前sim卡联系人已经开始读取，LOADICC_START
Contacts 收到loadicccontacts的广播，设置当前sim卡联系人已经读取完毕，LOADICC_FINISH
