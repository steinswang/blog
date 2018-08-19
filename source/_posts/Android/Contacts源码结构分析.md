---
title: Contacts源码结构分析
permalink: contacts-source-structure-analysis
categories:
  - Android
tags:
  - android
  - contacts
date: 2018-01-22 17:53:49
---

## 1.简介

联系人模块主要记录用户的联系人数据，方便用户快捷的操作和使用，主要包括本机联系人和Sim卡联系人。

本机联系人主要存储在手机内部存储空间，Android平台上是通过数据库进行存储，使用ContentProvider组件封装，提供复杂的字段用于表示联系人数据，并提供用户快捷的操作，比如增加，删除，修改，查询等等。

Sim卡联系人主要存储在Sim卡内部存储文件，包括adn、fdn、sdn。主要提供简单的字段用于表示联系人数据。并通过IccProvider提供的接口进行数据的增加、删除、修改、查询操作。

## 2.软件架构
联系人Contacts应用主要包括3个部分:
Contacts主要响应用户的请求和交互，数据显示。
ContactsProvider继承自Android四大组件之一的ContentProvider组件，封装了对底层数据库contact2.db的添删改查。
SQLite在底层物理性地存储了联系人数据。


主要交互流程如下图：

![](https://user-images.githubusercontent.com/35097187/44019032-3b7392de-9f10-11e8-8567-d4189730b46e.png)

Contacts模块的主要7块功能：

![](https://user-images.githubusercontent.com/35097187/44019045-3d496fe8-9f10-11e8-8077-89235449ccd4.png)

## 3. 各功能模块分析
### 3.1 联系人数据的显示
#### 3.1.1 联系人列表显示 ####

`简要说明：`

```xml
* PeopleActivity类负责联系人列表的显示。

* PeopleActivity包含4个Fragment，每个Fragment包含一个ListView。

* 各个Fragment中ListView的Adapter（BaseAdapter的子类）负责将数据填充到ListView。

* 各个Fragment的Loader类（CursorLoader的子类）负责加载数据。

* 实现LoadertManager接口负责管理这些CursorLoader。
```

![](https://user-images.githubusercontent.com/35097187/44019049-3dc1ee1e-9f10-11e8-89f3-14dac608ea42.png)

**为什么使用Loader？**
```xml
1. Loaders确保所有的cursor操作是异步的，从而排除了UI线程中堵塞的可能性。
2. 当通过LoaderManager来管理，Loaders还可以在activity实例中保持当前的cursor数据，也就是不需要重新查询（比如，当因为横竖屏切换需要重新启动activity时）。
3. 当数据改变时，Loaders可以自动检测底层数据的更新和重新检索。
```

**数据加载流程概览：**

![](https://user-images.githubusercontent.com/35097187/44019046-3d7130f0-9f10-11e8-945c-7a0eff02b65c.png)

**流程具体分析：**

先上图：
![](https://user-images.githubusercontent.com/35097187/44019033-3ba1bba0-9f10-11e8-94f9-e41053c967b4.jpeg)

进入Contacts应用，程序的主入口Activity是`PeopleActivity`。

进入`onCreate`方法：

`createViewsAndFragments(savedState);`

此方法创建视图和Fragments，进入此方法：

```java
mFavoritesFragment = new ContactTileListFragment();
mAllFragment = new DefaultContactBrowseListFragment();
mGroupsFragment = new GroupBrowseListFragment();
```

发现创建了3个Fragment，分别是 收藏联系人列表、所有联系人列表、群组列表。

进入`DefaultContactBrowseListFragment`：

发现`DefaultContactBrowseListFragment`的祖父类是：

`ContactEntryListFragment<T extends ContactEntryListAdapter>`

首先分析此基类：

发现此基类实现了`LoadManager`接口，实现了该接口3个重要的抽象方法：

```java
public Loader<D> onCreateLoader(int id, Bundle args);//创建Loader
public void onLoadFinished(Loader<D> loader, D data);//数据加载完毕后的回调方法
public void onLoaderReset(Loader<D> loader);//数据重新加载
```

该类同时提供了重要的抽象方法：

```java
protected abstract T createListAdapter();//创建适配器Adapter类。
```

这意味着,子类可以按需求创造自己的适配器Adapter类,完成各个子界面Listview的数据显示，如3.1节图1所示。

然后回到`DefaultContactBrowseListFragment`类：

在执行`onCreateView`之前，会执行父类的一些方法，顺序如下：

```java
onAttach()
setContext(activity);
setLoaderManager(super.getLoaderManager());
```

`setLoaderManager`中设置当前的LoaderManager实现类。 

加载联系人列表数据的过程中，这个类是`ProfileandContactsLoader`。

之后执行`onCreate`方法。

进入`DefaultContactBrowseListFragment`的`onCreate(Bundle)`方法：

```java
mAdapter = createListAdapter();
```

发现在这里创建了`ListAdapter`：

```java
DefaultContactListAdapter adapter = 
new DefaultContactListAdapter(getContext());
```

可以知道创建的`ListAdapter`类型是`DefaultContactListAdapter`并返回到`DefaultContactBrowseListFragment`类。

执行完`onCreate`方法之后，

执行`DefaultContactBrowseListFragment`的`onCreateView`方法。

进入DefaultContactBrowseListFragment的onCreateView方法：

```java
mListView = (ListView)mView.findViewById(android.R.id.list);
mListView.setAdapter(mAdapter);
```

首先获取了ListView用以填充联系人数据，然后设置了适配器，但是此时适配器中的数据是空的，直到后面才会加载数据更新UI。
在`onCreateView`方法执行完之后，在UI可见之前回调执行`Activity`的`onStart`方法。

进入`DefaultContactBrowseListFragment`的`onStart`方法：

```java
mContactsPrefs.registerChangeListener(mPreferencesChangeListener);
startLoading();
```

首先注册了一个`ContentObserv`e的子类监听数据变化。
然后执行`startLoading`方法，**目测这应当就是开始加载数据的方法了！**

进入`DefaultContactBrowseListFragment`的`startLoading`方法：

```java
int partitionCount = mAdapter.getPartitionCount();
for (int i = 0; i < partitionCount; i++) {
……
Partition partition = mAdapter.getPartition(i);
startLoadingDirectoryPartition(i);
……}
```

`Partition`这个类持有一个`Cursor`对象，用来存储数据。
`Adapter`持有的`Partition`，`Partition`类代表了当前需要加载的`Directory`，可以理解为一个联系人集合，比如说本地联系人、Google联系人……这里我们假设只加载本地联系人数据，所以`partitionCount=1`。

从这里我们可以做出猜测：
联系人数据不是想象中的分页（每次N条联系人数据）加载，也不是说一次性全部加载，而是一个账户一个账户加载联系人数据，加载完毕一个账户就在uI刷新并显示数据。

进入`DefaultContactBrowseListFragment`的`startLoadingDirectoryPartition`方法：

```java
loadDirectoryPartition(partitionIndex, partition);
```

进入此方法：

```java
getLoaderManager().restartLoader(partitionIndex, args, this);
```

这个方法是LoaderManager实现类的方法，参照文档解释：

>这个方法会新建/重启一个当前LoaderManager中的Loader，将回调方法注册给他，并开始加载数据。也就是说会回调LoaderManager的onCreateLoader()方法。
>Starts a new or restarts an existing android.content.Loader in this manager, registers the callbacks to it, and (if the activity/fragment is currently started) starts loading it

进入`LoadManager`接口的实现类：`LoaderManagerImpl`的`restartLoader`方法内部：

```java
LoaderInfo info = mLoaders.get(id);
Create info=
createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
//进入createAndInstallLoader方法：
LoaderInfo info = createLoader(id, args, callback);
installLoader(info);
//进入createLoader方法：
LoaderInfo info = new LoaderInfo(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
Loader<Object> loader = callback.onCreateLoader(id, args);
//关键方法出现了！LoadManager接口的抽象方法的onCreateLoader方法被回调了！
//然后installLoader方法启动了这个Loader！
info.start();
```

进入`ContactEntryListFragment`的`onCreateLoader`方法，位于`DefaultContactBrowseListFragment`的祖父类`ContactEntryListFragment`中：

```java
CursorLoader loader = createCursorLoader(mContext);//创建Loader
mAdapter.configureLoader(loader, directoryId);//配置Loader
```

发现在此方法中，首先调用`createCursorLoader`方法创建了`Loader`。
然后通过`configureLoader`方法配置`Loader`的`query`方法的查询参数，也就是配置SQL中select查询语句的参数。
这也同时意味着，`ContactEntryListFragment`类的子类们可以重写`createCursorLoader`方法以提供适合自身的Loader，重写`configureLoader`方法为Loader配置合适的参数，适配各种自定义的查询获取数据。

观察`createCursorLoader`方法在`DefaultContactBrowseListFragment`类中实现：

```java
return new ProfileAndContactsLoader(context);
```

直接返回了`DefaultContactBrowseListFragment`的数据加载器：`ProfileAndContactsLoader`
这就是`DefaultContactBrowseListFragment`的Loader实现类（数据加载器）。

然后再看一下`ProfileAndContactsLoader`类是如何加载数据的呢？
发现它继承自`CursorLoader`，而`CursorLoader`又继承自`AsyncTaskLoader<D>`
在关键的`LoadBackGround()`方法中：
异步调用了ContentResolver的`query`方法：

```java
Cursor cursor = getContext()
.getContentResolver()
.query(mUri, mProjection, mSelection,
                    mSelectionArgs, mSortOrder, mCancellationSignal);
cursor.registerContentObserver(mObserver);
```

通过这个Query方法，实现了对联系人数据的查询,返回Cursor数据。并绑定了数据监听器。


那么问题来了

```java
query(mUri, mProjection, mSelection,mSelectionArgs, mSortOrder, mCancellationSignal)
```
的这些参数那里指定的呢？
`configureLoader`方法在`DefaultContactListAdapter`类中实现，实现了对`query`参数的配置：

```java
configureUri(loader, directoryId, filter);
loader.setProjection(getProjection(false));
configureSelection(loader, directoryId, filter);
loader.setSortOrder(sortOrder);
```

可以看到，配置了Loader主要的几个参数：Uri，Projection，Selection，SortOrder。
这些参数用于最后和ContactsProvider交互的方法Query方法中……

最终查询`ContactsProvider2`的uri是：

```xml
Uri：content://com.android.contacts/contacts?address_book_index_extras=true&directory=0
```

发现ContentProvider的服务类似一个网站，uri就是网址，而请求数据的方式类似使用Get方式获取数据。

最后通过ContentProvider2构建的查询语句是这样的：

```sql
SELECT 
_id, display_name, agg_presence.mode AS contact_presence, 
contacts_status_updates.status AS contact_status, photo_id, photo_thumb_uri, lookup, 
is_user_profile 
FROM view_contacts 
LEFT OUTER JOIN agg_presence ON (_id = agg_presence.presence_contact_id) LEFT OUTER JOIN 
status_updates contacts_status_updates ON
(status_update_id=contacts_status_updates.status_update_data_id)
```
可以发现最后通过ContactsProvider2实现的查询，并不是直接查询相关的表（Contacts表、rawcontacts表，data表……），而是直接查询`view_contacts`视图，因为这样会有更加高的效率。
这也就意味着如果想给联系人数据库新增一个字段供界面使用，仅修改对应的表结构是不行，还要修改对应的视图才能得到想要的效果。


查询完毕后，回调`LoaderManager`的`onLoadFinished`方法，完成对UI界面的更新：

```java
onPartitionLoaded(loaderId, data);
```

接着进入`onPartitionLoaded`方法：

```java
mAdapter.changeCursor(partitionIndex, data);
```

进入这个`changeCursor`方法：

```java
mPartitions[partition].cursor = cursor;
notifyDataSetChanged();
```

发现在这里改变了`Adapter`的数据集`Cursor`，并发出通知数据已经改变，UI进行更新。

至此，默认联系人数据的显示分析到此结束。

其他`Fragment`的数据填充基本仍然类似此流程，所不同的只是各自的`Fragment`、`Adapter`、`CursorLoader`以及`CursorLoader`配置的参数（uri，projection,selection,args,order……）有所不同。

可以参考下表：

| Fragment | Adapter | CursorLoader |
| :------: | :------: | :------: |
|DefaultContactBrowseListFragment(默认联系人列表)|DefaultContactListAdapter|ProfileAndContactsLoader|
|ContactTitleListFragment(收藏联系人列表)|ContactTileAdapter|ContactTileLoaderFactory StarredLoader|
|ContactTitleFrequentFragment(常用联系人列表)|ContactTitleAdapter|ContactTileLoaderFactory FrequentLoader|
|GroupBrowseListFragment(群组列表)|GroupBrowseLIstAdapter|GroupListLoader|
|GroupDetailFragment(指定ID群组的联系人列表)|GroupMemberTileAdapter|GroupMemberLoader|
|ContactDetailFragment(指定ID联系人信息)|ViewAdapter|ContactLoader| 

#### 3.1.2 联系人详细信息数据的显示 ####
**关键类：**
```java
ContactDetailActivity

ContactDetailFragment  

ContactLoaderFragment //不可见 负责加载联系人详细数据，集成LoadManager对象。

ContactLoader   //联系人详细信息Loader。

ContactDetailLayoutController     //布局控制类。
```

原理类似列表显示，如下简要说明： 
```xml
* ContactLoaderFragment类创建了一个实现LoaderManager.LoaderCallbacks<Contact>接口的对象，数据类型指定为Contacts。负责创建、管理ContactLoader。
 
* 得到当前用户选择的联系人URI，配置对应的ContactLoader。 

* 后台数据查询完毕后，回调LoadManager的onLoadFinished()方法，并将数据以Contacts的数据类型返回，然后回调ContactDetailLoaderFragmentListener的onDetailsLoaded()方法。 

* onDetailsLoaded()方法中，新开一个线程，通过ContactDetailLayoutController类的setContactData(Conatct)设置数据，刷新ContactDetailFragment。
```

### 3.2 联系人数据的编辑和存储

#### 3.2.1 编辑界面相关####

联系人数据所属的账号不同，加载的UI也是不同的，比如Sim卡联系人一般只有name，phone num，但是本地账号联系人可能就会有email，

address，website等信息…… 

联系人数据UI的加载是通过代码动态加载的，而不是xml文件写死的。

那么问题来了， 

新建联系人的界面是如何设计？ 

![](https://user-images.githubusercontent.com/35097187/44019034-3bc9de0a-9f10-11e8-8f38-f764c64cc8e2.jpeg)

先进入新建联系人界面：

主界面`PeopleActivity`中点击新建联系人Button，触发`onOptionsItemSelected`方法中的

case R.id.menu_add_contact分支： 

执行`startActivity(intent);` 

`startActivity`启动Intent，Intent的Action设置为android.intent.action.INSERT 

找到匹配此Action的Activity：`ContactEditorActivity`

`ContactEditorActivity`的布局文件：
 
`ContactEditorActivity`的`onCreate()`方法中找到布局： 

`setContentView(R.layout.contact_editor_activity);`

在xml文件中找到这个布局：

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
 <fragment class="com.android.contacts.editor.ContactEditorFragment"
            android:id="@+id/contact_editor_fragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
</FrameLayout>
```

只包含一个Fragment：`ContactEditorFragment`。程序解析Xml文件到这里就会执行`ContactEditorFragment`类。

进入`ContactEditorFragment`的`onCreateView`方法：

```xml
//展开布局 
final View view
= inflater.inflate(R.layout.contact_editor_fragment, container, false);    
//找到布局中的一个线性布局
//关键的布局是contact_editor_fragment中的一个iD为editors的线性布局！
mContent = (LinearLayout) view.findViewById(R.id.editors);
```

找到`contact_editor_fragment`：

```xml
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fillViewport="true"
    android:fadingEdge="none"
    android:background="@color/background_primary"
>
    <LinearLayout android:id="@+id/editors"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
    />
</ScrollView>
```

于是确认`ContactEditorFragment`的根布局就是一个id为`editors`的LinearLayout。 
想到上一步的语句：

```xml
mContent = (LinearLayout) view.findViewById(R.id.editors);
```

所以关键就在于，接下来在代码中为mContent这个线性布局动态添加地了什么UI，而这些UI才是真正显示的东西。

`ContactEditorFragment`的`onCreateView`方法执行完毕之后，会调用`onActivityCreate()`方法：

```java
if (Intent.ACTION_INSERT.equals(mAction)) {
final Account account = mIntentExtras == null ? null : (Account) 
mIntentExtbindEditorsForNewContactras.getParcelable(Intents.Insert.ACCOUNT);
final String dataSet = mIntentExtras == null ? null :
                        mIntentExtras.getString(Intents.Insert.DATA_SET);
if (account != null) {
// Account specified in Intent
createContact(new AccountWithDataSet(account.name, account.type, dataSet));
}
```

上面代码首先取出了当前Account信息，数据信息。封装为一个`AccountWithDataSet`对象，作为`createContact`方法的参数。之前我们分析过，编辑界面和账户是高度相关的，所以对UI的动态操作必然和Account对象相关。进入`createContact`方法。

看一下`ContactEditorFragment`中的`createContact()`到底对界面干了什么！！ 

`createContact`方法中调用了`bindEditorsForNewContact(account, accountType)`: 

关键代码：
```java
……
final RawContact rawContact = new RawContact();
if (newAccount != null) {
    rawContact.setAccount(newAccount);
} else {
    rawContact.setAccountToLocal();
}
final ValuesDelta valuesDelta = ValuesDelta.fromAfter(rawContact.getValues());
final RawContactDelta insert = new RawContactDelta(valuesDelta);
……
mState.add(insert);
bindEditors();
```

发现暂时还是没有对界面做什么事情，任然处于酝酿阶段……

首先使用传入的Accout对象创建一个`RawContact`对象，然后使用`RawContact`对象构建了一个`RawContactDelta`对象insert，接着就将insert对象放入`RawContactDeltaList` 对象`mState`中。

```xml
RawContact类：raw contacts数据表内的一条数据，表示一个联系人某一特定帐户的信息。存储Data表中一些数据行（电话号码、Email、地址……）的集合及一些其他的信息。
他的存储结构为： HashMap<String, ArrayList<ValuesDelta>>

RawContactDelta类：包含RawContact对象（即一个联系人某一特定帐户的信息），并具有记录修改的功能。

RawContactDeltaList类：内部的存储结构是ArrayList<RawContactDelta>，可以理解为 单个联系人所有账户的数据集合。
```

然后调用了`bindEditors()`法。 

关键代码如下：
```java
……
mContent.removeAllViews();
……
final BaseRawContactEditorView editor;
……
editor = (RawContactEditorView) inflater.inflate(R.layout.raw_contact_editor_view,mContent, false);
//添加视图了……………………
mContent.addView(editor);
//为自定义视图BaseRawContactEditorView设置状态，必然是修改UI的操作！
editor.setState(rawContactDelta, type, mViewIdGenerator, isEditingUserProfile());
```

可以看到，`mContent`这个LinearLayout添加的View是`editor`，而`editor`是一个自定义的视图`BaseRawContactEditorView`，布局是`R.layout.raw_contact_editor_view`。

找到`raw_contact_editor_view`布局，发现该布局包含新建联系人页面所有的UI：

![](https://user-images.githubusercontent.com/35097187/44019036-3bf3c4ae-9f10-11e8-81b0-dfa54fe8fad9.jpeg) 

```xml
<com.android.contacts.editor.RawContactEditorView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:paddingTop="@dimen/editor_padding_top">
<include
用户账户相关UI
        layout="@layout/editor_account_header_with_dropdown" />
    <LinearLayout
        android:id="@+id/body"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">
        <LinearLayout
            android:layout_height="wrap_content"
            android:layout_width="match_parent"
            android:orientation="horizontal"
            android:paddingTop="8dip">
            <LinearLayout
                android:layout_height="wrap_content"
                android:layout_width="0dip"
                android:layout_weight="1"
                android:orientation="vertical">
                <include
            Name相关的UI
                    android:id="@+id/edit_name"
                    layout="@layout/structured_name_editor_view" />
                <include
            拼音名
                    android:id="@+id/edit_phonetic_name"
                    layout="@layout/phonetic_name_editor_view" />
            </LinearLayout>
            <include
            照片相关的UI
                android:id="@+id/edit_photo"
                android:layout_marginRight="8dip"
                android:layout_marginEnd="8dip"
                layout="@layout/item_photo_editor" />
        </LinearLayout>
        <LinearLayout
            中间部分Item的显示在此处
            android:id="@+id/sect_fields"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:layout_marginBottom="16dip"/>
            添加其他字段 按钮
        <Button
            android:id="@+id/button_add_field"
            android:text="@string/add_field"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:layout_marginBottom="32dip"/>
    </LinearLayout>
</com.android.contacts.editor.RawContactEditorView>
```

1.那么问题来了：中间的那部分布局（电话、地址……）去哪儿了？

搜索有可能包含这些内容的线性布局`sect_fields`，发现在`RawContactEditorView`类中初始化为`mFields`：

`mFields = (ViewGroup)findViewById(R.id.sect_fields);`

那么只需要看代码中对mFields添加了什么UI！

2.回到之前的`bindEditors()`方法，`RawContactEditorView` 对象`editor`从xml中解析完成后，执行了`setState`方法：

```java
editor.setState(rawContactDelta, type, mViewIdGenerator, isEditingUserProfile());
```

1.`进入RawContactEditorView`类，找到`setState`方法：

```java
public void  setState(RawContactDelta state, AccountType type, ViewIdGenerator vig,boolean isProfile)
……
// 遍历当前账户所有可能的item种类，如电话，姓名，地址……，并分别创建自定义视图KindSectionView
   for (DataKind kind : type.getSortedDataKinds()) {
……
  final KindSectionView section = (KindSectionView)mInflater.inflate(
                        R.layout.item_kind_section, mFields, false);
                section.setEnabled(isEnabled());
                section.setState(kind, state, false, vig);
                mFields.addView(section);
……
}
```

发现遍历了当前账号类型中所有可能的数据类型（`DataKind`），

创建了相关的自定义视图`KindSectionView`对象`section`，

再将`section`对象添加到`mFields`中显示，

这个mFields正是之前在`RawContactEditorView`类中初始化的线性布局：
```java
mFields = (ViewGroup)findViewById(R.id.sect_fields)。
```

到这里，基本可以确定，中间部分（也就是除了Name、Photo 和底部的添加字段Button之外的部分），就是通过这个`mFields`动态的根据当前账户类型添加编辑的`KindSectionView`条目来填充的。

首先观察一下`KindSectionView`的布局文件`item_kind_section`：

```xml
<com.android.contacts.editor.KindSectionView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">
    <include                   这是一个TextView，title
        android:id="@+id/kind_title_layout"
        layout="@layout/edit_kind_title" />
      <LinearLayout            线性布局，用于添加EditText
        android:id="@+id/kind_editors"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical" />
    <include                   添加新条目的TextView，初始化状态不可见
        android:id="@+id/add_field_footer"
        layout="@layout/edit_add_field" />
</com.android.contacts.editor.KindSectionView>
```

1.`KindSectionView`加载完xml文件之后，会执行`onFinishInflate`方法：

```java
mTitle = (TextView) findViewById(R.id.kind_title);
mEditors = (ViewGroup) findViewById(R.id.kind_editors); 
mAddFieldFooter = findViewById(R.id.add_field_footer);
```

把Xml文件中三个主要的部分都得到了，接下来重点就是观察代码中对他们做了什么。

在第12步中，加载完xml文件之后，执行`KindSectionView`的`setState`方法：

```java
section.setState(kind, state, false, vig);
```

将`rawContactDelta`对象`state`传递给了`KindSectionView`类的`setState`方法：

进入`KindSectionView`类的`setState`方法：

```java
mKind = kind;
mState = state;
rebuildFromState();
```

先进行局部变量的赋值

1.然后进入到`rebuildFromState()`方法：
```java
  for (ValuesDelta entry : mState.getMimeEntries(mKind.mimeType)) {
       //……遍历当前账户可能的键值对，比如电话、Email、地址……
      createEditorView(entry);  //这个方法应当是创建EditText的方法！
  }
```

在这个方法中，对`mState`集合中所有Mime类型的`ValuesDelta`集合（`ArrayList<ValuesDelta>`类型）进行遍历，而后将每一个 `ValuesDelta`对象 `entry`作为参数调用了`createEditorView(entry)`也就是创建各个种类的`EditText`方法，根据`entry`对象创建相应的`EditText`！

简单说，就是创建`mState`中存在的类型的`EditText`。
当然……这还都只是猜测，需要进入`createEditorView`方法确认。

1.进入`createEditorView`方法：

```java
view = mInflater.inflate(layoutResId, mEditors, false);
Editor editor = (Editor) view;
editor.setValues(mKind, entry, mState, mReadOnly, mViewIdGenerator);
```

第13步初始化的`mEditors`对象（也就是那个被猜测应该是放`EditText`的线性布局）在这里被使用！

1.联系上下文，实际上此时editor对象是`TextFieldsEditorView`类的对象，进入`TextFieldsEditorView`的`setValues`方法，看看他是如何根据entry对象创建`EditText`的：

```java
public void setValues(DataKind kind, ValuesDelta entry, RawContactDelta state, boolean readOnly,ViewIdGenerator vig) {
int fieldCount = kind.fieldList.size();  //获取所有可能的datakind的总数
for (int index = 0; index < fieldCount; index++)    //遍历所有可能的datakind，
{ 
final EditText fieldView = new EditText(mContext);  //创建EditText对象，之后进行配置
fieldView.setLayoutParams……
fieldView.setTextAppearance(getContext(), android.R.style.TextAppearance_Medium);
fieldView.setHint(field.titleRes);   //EditText的Hint
……     

fieldView.addTextChangedListener(new TextWatcher()  //注册TextChangedListener
{
	@Override
	public void afterTextChanged(Editable s) {
	    // Trigger event for newly changed value
	    onFieldChanged(column, s.toString());
	}
	mFields.addView(fieldView);    //将EditText添加到当前的线性布局中！
}
```
注释基本解释了如何通过一个`ValuesDelta`（理解为键值对集合）对象`entry`创建布局中的所有`EditText`。

至此，联系人编辑界面的显示原理基本分析完成。


数据存储相关

### 3.3 Sim联系人数据的整合
Sim卡联系人数据的显示
开机自动导入Sim卡联系人
telephony中IccProvider浅析
Sim卡联系人的手动导入导出

### 3.4 SD卡备份/恢复联系人
- SD卡备份/恢复联系人
- 联系人数据导出到SD卡

### 3.5 联系人搜索

### 3.6 Google联系人同步

### 3.7 其他零碎功能

转自：https://blog.csdn.net/Kafka_88/article/details/50670406

