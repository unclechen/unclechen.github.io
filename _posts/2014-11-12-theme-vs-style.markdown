---
layout: post
title: Theme vs Style
date: '2014-11-12 16:28:02'
cover_image: '/content/images/2014/11/theme-vs-style.png'
---

Android 5.0 Lollipop brings with it new functionality which allows you to specify an override theme for a View (and any descendents). Let's have a look at how and why you would use it.

## Why?

You've probably already been using this functionality for a while without knowing: `Theme.Holo.Light.DarkActionBar`.

Think about the Light.DarkActionBar theme for a second. It's a light theme for your content (background is light and the foreground is dark), but the action bar use a dark theme (dark background with light foreground color).

Without being able to supply a seperate theme, you would need to manually set the text color and other foreground colors to some sort of inverse. Eugh.

That is where the old `actionBarWidgetTheme` attribute came in, it allowed you to specify a theme to be used **only** for your action bar. Here's an excerpt from the platform DarkActionBar theme:

{% highlight xml %}
<style name="Theme.Holo.Light.DarkActionBar">
    <item name="android:actionBarWidgetTheme">@android:style/Theme.Holo</item>
</style>
{% endhighlight %}

Thus making your action bar using the dark theme.

### Underlying functionality

So how do this work under-the-covers? Simple, it's the [ContextThemeWrapper](https://developer.android.com/reference/android/view/ContextThemeWrapper.html) class which has been available since API v1. It's a pretty simple class and the clue to what it does is in the name:

It wraps an existing Context (say your Activity), and then  **overlays** a new theme on top of that Context's theme. This is important to understand as this leads on to...

## ThemeOverlay

You may have seen these themes in the Lollipop SDK. There are two main overlays:

 * ThemeOverlay.Material.Light
 * ThemeOverlay.Material.Dark

So what exactly are these? Well again, the clue is in the name, and they are directly match to how ContextThemeWrapper works.

They are special themes which overlay the normal `Theme.Material` themes, overwriting relevant attributes to make them either light/dark.

### ThemeOverlay + ActionBar

The keen eyed of you will also have seen the ActionBar ThemeOverlay derivatives:

 * ThemeOverlay.Material.Light.ActionBar
 * ThemeOverlay.Material.Dark.ActionBar
 
These should only be used with the Action Bar via the new `actionBarTheme` attribute, or directly set on your [Toolbar](https://developer.android.com/reference/android/widget/Toolbar.html) (see below).

The only things these currently do differently to their parents is that they change the `colorControlNormal` to be `android:textColorPrimary`, thus making any text and icons opaque.

## android:theme

Let's get back to the new Lollipop functionality. As mentioned you can now specify a theme directly onto a View in your layout.  The most common use of this will (probably) be using Toolbar:

{% highlight xml %}
<Toolbar  
    android:layout_height="?android:attr/actionBarSize"
    android:layout_width="match_parent"
    android:background="?android:attr/colorPrimaryDark"
    android:theme="@android:style/ThemeOverlay.Material.Dark.ActionBar" />
{% endhighlight %}

Hopefully you can now see how it all comes together. We have now made the Toolbar have a dark theme, ensuring that it's content is light in color and has contrast againt the dark background.

One thing to note is that `android:theme` in Lollipop propogates to all children declared in the layout:

{% highlight xml %}
<LinearLayout
    android:theme="@android:style/ThemeOverlay.Material.Dark">
    
    <!-- Anything here will also have a dark theme -->
    
</LinearLayout>
{% endhighlight %}

Your children can set their own theme if needed.

## Example

Let's wrap this up with a question that I've been asked recently:

> How do I set android:colorEdgeEffect so that it only takes effect on a single view?

The `colorEdgeEffect` attribute is a new theme attribute in Android 5.0, which is used to customize the color of the overscroll effect for lists, etc.

As this is a theme attribute you can not just set it directly on the view. Instead we need to use `android:theme` with a custom ThemeOverlay. Our custom overlay just sets `android:colorEdgeEffect` to be red. We then set this theme on to the view so that it takes effect.

**res/values/themes.xml**

{% highlight xml %}
<style name="RedThemeOverlay" parent="android:ThemeOverlay.Material">
    <item name="android:colorEdgeEffect">#FF0000</item>
</style>
{% endhighlight %}

**res/layout/fragment_list.xml**

{% highlight xml %}
<ListView
    ...
    android:theme="RedThemeOverlay" />
{% endhighlight %}

Just to note, colorEdgeEffect was just an example here, this technique can be used with **all** theme attributes.

## Theme vs Style

So what exactly is the difference? Well they are both declared in exactly the same way (which you already know), the difference comes in how they're used.

Themes are meant to be the global source of styling for your app. The new functionality doesn't change that, it just allows you to tweak it per view.

Styles are meant to be applied at a view level. Internally, when you set `style` on a View, the LayoutInflater will read the style and apply it to the [AttributeSet](https://developer.android.com/reference/android/util/AttributeSet.html) before any explicit attributes (this allows you to override style values on a view).

Values in an attribute set can reference values from the View's theme.

TL;DR: Themes are global, styles are local.

## AppCompat

So how does AppCompat fit into this? Well it obviously backports some of the new color theme attributes.

It also backports the `android:theme` functionality for certain widgets, currently **only** `android.support.v7.widget.Toolbar`.

See my [AppCompat v21](https://chris.banes.me/2014/10/17/appcompat-v21/) blog post for more information.