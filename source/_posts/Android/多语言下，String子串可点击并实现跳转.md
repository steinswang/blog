---
title: 多语言下，String子串可点击并实现跳转
permalink: substring-jump
categories:
  - Android
tags:
  - android
date: 2018-01-22 17:53:49
---

## 总结
通过Spannable实现色彩效果，ClickableSpan实现点击。

对于resource string，通过添加标签位，来计算不同语言下，可点击的substring的index

## 工具类

```java
//LinkifyUtils.java
public class LinkifyUtils {
    private static final String PLACE_HOLDER_LINK_BEGIN = "LINK_BEGIN";
    private static final String PLACE_HOLDER_LINK_END = "LINK_END";

    private LinkifyUtils() {
    }

    /** Interface that handles the click event of the link */
    public interface OnClickListener {
        void onClick();
    }

    /**
     * Applies the text into the {@link TextView} and part of it a clickable link.
     * The text surrounded with "LINK_BEGIN" and "LINK_END" will become a clickable link. Only
     * supports at most one link.
     * @return true if the link has been successfully applied, or false if the original text
     *         contains no link place holders.
     */
    public static boolean linkify(TextView textView, StringBuilder text,
                                  final OnClickListener listener) {
        // Remove place-holders from the string and record their positions
        final int beginIndex = text.indexOf(PLACE_HOLDER_LINK_BEGIN);
        if (beginIndex == -1) {
            textView.setText(text);
            return false;
        }
        text.delete(beginIndex, beginIndex + PLACE_HOLDER_LINK_BEGIN.length());
        final int endIndex = text.indexOf(PLACE_HOLDER_LINK_END);
        if (endIndex == -1) {
            textView.setText(text);
            return false;
        }
        text.delete(endIndex, endIndex + PLACE_HOLDER_LINK_END.length());

        textView.setText(text.toString(), TextView.BufferType.SPANNABLE);
        textView.setMovementMethod(LinkMovementMethod.getInstance());
        Spannable spannableContent = (Spannable) textView.getText();
        ClickableSpan spannableLink = new ClickableSpan() {
            @Override
            public void onClick(View widget) {
                listener.onClick();
            }

            @Override
            public void updateDrawState(TextPaint ds) {
                super.updateDrawState(ds);
                ds.setUnderlineText(false);
            }
        };
        spannableContent.setSpan(spannableLink, beginIndex, endIndex,
                Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
        return true;
    }
}
```

## UI界面

```java
//XXXActivity.java
        StringBuilder contentBuilder = new StringBuilder();
        contentBuilder.append(getText(R.string.no_internet));
        LinkifyUtils.linkify(tvEmpty, contentBuilder, new LinkifyUtils.OnClickListener() {
            @Override
            public void onClick() {
                Intent intent = new Intent();
                intent.setClassName("com.android.settings", "com.android.settings.wifi.WifiSettings");
                startActivity(intent);
            }
        });
```

```xml
<string name="no_internet">No internet connection. Make sure  <xliff:g id="link_begin">LINK_BEGIN</xliff:g>Wi-Fi<xliff:g id="link_end">LINK_END</xliff:g> is turned on, then try again.</string>

<string name="no_internet">未连接到互联网。请确保  <xliff:g id="link_begin">LINK_BEGIN</xliff:g>Wi-Fi<xliff:g id="link_end">LINK_END</xliff:g> 网路已开启，然后重试。</string>

```
