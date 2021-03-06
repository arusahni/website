.. link: 
.. description: 
.. tags: Android, code
.. wordpress_id: 72
.. date: 2011-08-13 15:54:41
.. slug: action-bar-and-you
.. title: Action Bar and You!
.. description: In which I walk through adding an ActionBar to an Android app via GreenDroid.
.. comments: true

.. role:: strike
    :class: strike

So I've been developing an Android application on and off for the past few months. In the latest iteration of the app, I wanted to make some options that were hidden in a menu more discoverable. The accepted way of doing this in Android is through adopting the Action Bar design pattern. As described in the official `Android Developers Blog entry <http://android-developers.blogspot.com/2010/05/twitter-for-android-closer-look-at.html>`_ on the matter:

    The Action bar gives your users onscreen access to the most frequently used actions in your application. We recommend you use this pattern if you want to dedicate screen real estate for common actions. Using this pattern replaces the title bar. It works with the Dashboard, as the upper left portion of the Action bar is where we recommend you place a quick link back to the dashboard or other app home screen.
    
Awesome, right? A unified UI paradigm for Android. Let me just add the widget to my Activity... umm, uhh... where is it? Nowhere in the Android 2.x SDKs. Instead, Google decided to kill two birds with one stone and encouraged developers to look at `the source code <https://code.google.com/p/iosched/>`_ for the Google I/O schedule app. This way people could not only learn how to use and implement the pattern, but they can also see what a well-written Android app is supposed to look like.

That's all well and good, but what about the :strike:`lazy` efficient among us, who don't want to reinvent the wheel and write the same instrumentation every time we make an Activity? Once again, Google to the rescue. With the advent of library projects, the ADT plugin for Eclipse makes `importing standalone/reusable components into your existing project easy <http://developer.android.com/guide/developing/projects/projects-eclipse.html#ReferencingLibraryProject>`_. After this feature was made available, the number of rich, third-party libraries for Android skyrocketed. Among these were `several <http://actionbarsherlock.com/>`_ that had `reusable <https://github.com/johannilsson/android-actionbar/>`_ Action Bar controls. After some experimentation, I settled on `GreenDroid <https://github.com/cyrilmottier/GreenDroid>`_.

So, GreenDroid
--------------

GreenDroid's claim to fame is the notion that you need to alter very little of your existing code to get attractive, functional UI components. Since my project was mostly complete at the time of implementation, this was a huge plus. And, for the most part, it ended up being true.

.. TEASER_END

After I added GreenDroid as a library project to my existing app, I had to instruct Android to load GD's theme. That was as simple as adding the "android:theme" attribute to my AndroidManifest.xml:

.. code:: xml

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher" android:name="MyAwesomeApp" android:description="@string/app_description" android:theme="@style/Theme.GreenDroid.NoTitleBar">
    
Next, I had to alter my main Application class to extend the GDApplication class. This wasn't a problem as I was already extending Application for some global variables.

.. code:: java

    public class MyAwesomeApp extends GDApplication {
    	@Override
    	public Class getHomeActivityClass() {
    		return HomeActivity.class;
    	}
    }
    
As you can see, there's an interesting override happening here - `getHomeActivityClass`.  This method is used by GreenDroid to determine which Activity to launch when the user taps the home icon on the left side of the ActionBar.  Since my "home" activity is usually the second one on the Activity stack, this is an awesome feature to have.

That was painless, right?  I then moved on to port my Activities.  My home activity needed an ActionBar with a refresh button:

.. code:: java
    :number-lines:
    
    public class HomeActivity extends GDActivity { //Needed in order to implement the ActionBar
    	private LoaderActionBarItem throbber;
    	private final int REFRESH = 0;
    
    	public void onCreate(Bundle savedInstanceState) {
    		super.onCreate(savedInstanceState);
    		setActionBarContentView(R.layout.home_activity); //NB: NOT setContentView()
    		initActionBar();
    	}
    
    	private void initActionBar() {
    		throbber = (LoaderActionBarItem) addActionBarItem(Type.Refresh, REFRESH);
    	}
    	
    	@Override
    	public boolean onHandleActionBarItemClick(ActionBarItem item, int position) { 
    		switch (item.getItemId()) {
    		case REFRESH:
    			refreshContent();
    			break;
    		default:
    			super.onHandleActionBarItemClick(item, position);	
    		}
    		return true;
    	}
    
    	private void refreshContent() {
    		//do stuff. If this is an AsyncTask, move the line below into onPostExecute
    		throbber.setLoading(false); //switch back to the static refresh image
    	}
    }
    
There's a lot going on there.  Breaking it down...

* Line 2: I'm storing a reference to the ActionBarItem so I can toggle the indeterminate progress (busy) state from anywhere within the Activity. This is needed to revert back to the refresh button when the content is done refreshing.
* Line 3: Just as one would do when defining result codes for a spawned Activity, GreenDroid requires that each ActionBarItem be associated with an integer for the purposes of action identification.
* Line 12: Here I'm appending the action to the ActionBar, defining the action ID, and assigning it to my Activity-level variable for future reference.

Run it and what do we get?  

.. figure:: http://i.imgur.com/CydMXH2l.png
   :alt: ActionBar screenshot.
   :target: http://imgur.com/CydMXH2
   
   BOOM. Easy. 

The next Activity, however, wasn't as easy. It's a `ListActivity`, which, at first glance, didn't seem to be a challenge.  However, there was a complication due to the way layouts are built by `GDLIstActivity`.  According to the creator of GreenDroid, Cyril Mottier:

    The setActionBarContentView is normally only a helper method for the GDActivity. If you look at the ActionBarActivity interface, you'll see this method is not a part of the ActionBarActivity interface
    
    The correct method to use when you want to have a custom layout in GDListActivity is to override the createLayout method.


So, I couldn't just replace the setContentView call as I did before.  Instead I needed to do two things: override the default layout used by GDListActivity, and alter the original layout to provide the components that GreenDroid requires.  Overriding the default layout was pretty straightforward...


.. code:: java
    :number-lines: 1
    
    public class CustomList extends GDListActivity {
    	public void onCreate(Bundle savedInstanceState) {
    		super.onCreate(savedInstanceState);
    		initActionBar();
    	}
    	@Override
    	public int createLayout() {
    		return R.layout.custom_list;
    	}
    }
    
... but the updated layout caused a little head scratching.  After some experimentation and inspection of the code in the `GDCatalog app <https://github.com/cyrilmottier/GreenDroid/tree/master/GDCatalog>`_, I came up with the following.

.. code:: xml

    <?xml version="1.0" encoding="utf-8"?>
    <greendroid.widget.ActionBarHost
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@id/gd_action_bar_host"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:orientation="vertical">
        <greendroid.widget.ActionBar
            android:id="@id/gd_action_bar"
            android:layout_height="@dimen/gd_action_bar_height"
            android:layout_width="fill_parent"
            android:background="?attr/gdActionBarBackground" />
        <FrameLayout
            android:id="@id/gd_action_bar_content_view"
            android:layout_height="0dp"
            android:layout_width="fill_parent"
            android:layout_weight="1"
            android:background="@color/gray">
            <!-- INSERT ORIGINAL LAYOUT HERE -->
            <ListView
                android:layout_height="fill_parent"
                android:id="@android:id/list"
                android:layout_weight="1"
                android:layout_width="fill_parent"
                android:cacheColorHint="@color/gray">
            </ListView>
        </FrameLayout>
    </greendroid.widget.ActionBarHost>
    
Of note should be the use of a FrameLayout to wrap the content of the original layout.  The IDs for the ActionBarHost, ActionBar and FrameLayout _must_ must not differ from those used above.

And, with that, I added action bars to my interface!