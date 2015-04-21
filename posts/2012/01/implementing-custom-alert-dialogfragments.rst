.. link: 
.. description: 
.. tags: Android, code
.. date: 2012-01-15 11:30:10
.. slug: implementing-custom-alert-dialogfragments
.. title: Implementing Custom Alert DialogFragments
.. description: In which I detail how to display alert dialogs in Android utilizing the system's button order and styling.
.. comments: true
.. wordpress_id: 94

.. role:: strike
    :class: strike
    
.. role:: java(code)
    :language: java

So, I've been working on further revisions to `my app <https://market.android.com/details?id=com.thirteenbytes.kongforums>`_.  The most notable of these revisions involves switching action bar implementations from `GreenDroid <https://github.com/cyrilmottier/GreenDroid>`_ to `ActionBarSherlock <http://actionbarsherlock.com/>`_.  While this will benefit me tremendously as I'll be able to use native Android 3.0+ components when running on those platforms, it means that I need to start using Fragments as well as the rest of the `Android Support Package <http://developer.android.com/sdk/compatibility-library.html>`_.

.. figure:: http://i.imgur.com/5nDrHwFl.jpg
    :alt: Steve Ballmer
    :target: http://imgur.com/5nDrHwF

    Refactor Refactor Refactor!
    
I've be saving much of what I have learned for another post, but I felt like sharing this tidbit right now.  One thing that quickly became apparent was that a custom dialog that I was using (with positive/negative buttons) was not going to cut it, as the system dialogs were rendering the button strip differently...

.. figure:: http://i.imgur.com/hKUvClPl.png
    :alt: A dialog
    :target: http://imgur.com/hKUvClP

    The button strip isn't displayed correctly
    
.. TEASER_END

This is implemented via...

.. code:: xml

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:minWidth="280dip"
        android:padding="10dip"
        android:id="@+id/layoutPageDialogRoot"
        >
        
        <EditText
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:id="@+id/editTextPageNumber"
            android:inputType="number"
            />
            
        <TextView
            android:id="@+id/textViewMaxPages"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content" android:text=""
            android:gravity="right"
            >
            <requestFocus></requestFocus>
        </TextView>
        
        <LinearLayout
            android:layout_height="wrap_content"
            android:id="@+id/linearLayout1"
            android:layout_width="fill_parent"
            >
            
                <Button android:text="Go!"
                    android:id="@+id/buttonOk"
                    android:layout_height="wrap_content"
                    android:layout_weight="1"
                    android:layout_marginTop="3dip"
                    android:layout_width="0dip"
                    />
                    
                <Button
                    android:text="Nevermind"
                    android:id="@+id/buttonCancel"
                    android:layout_height="wrap_content"
                    android:layout_weight="1"
                    android:layout_marginTop="3dip"
                    android:layout_width="0dip"
                    />
                    
        </LinearLayout>
    </LinearLayout>
    
While this may seem like a minor UI issue at first, there's something more to it - Android versions 3.0 and above switched the order of the default dialog button actions, placing the positive button on the right and the negative on the left.  The dialog above is using the pre-2.0 order.  When you're dealing with muscle memory, such UX concerns become important (especially for deletion dialogs).

So, when refactoring the :java:`PaginationDialog` into its own :java:`DialogFragment`, I decided to do more than port my code.  I realized that the native :java:`AlertDialog` class uses the operating system order, and is styled to match the OS version's conventions.  If only there was a way I could throw my layout into that... wait! There is - :java:`AlertDialog.Builder()` has a :java:`setView()` method!

By removing the buttons from the layout XML file and manually inflating it, I can stuff my Views into the dialog and piggyback off its functionality.

My buttonless layout...

.. code:: xml

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/layoutPageDialogRoot"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:minWidth="280dip"
        android:orientation="vertical"
        android:padding="10dip"
        >
    
        <EditText
            android:id="@+id/editTextPageNumber"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:inputType="number"
            />
    
        <TextView
            android:id="@+id/textViewMaxPages"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:gravity="right"
            android:text=""
            >
            <requestFocus></requestFocus>
        </TextView>
    
    </LinearLayout>
    
.. code:: java
    :number-lines: 1
    
    public class PaginationDialogFragment extends DialogFragment {
    
    	int currentPage, maxPages;
    
    	static PaginationDialogFragment newInstance(int currentPage, int maxPages) {
    		PaginationDialogFragment p = new PaginationDialogFragment();
    		Bundle args = new Bundle();
    		args.putInt("currentPage", currentPage);
    		args.putInt("maxPages", maxPages);
    		p.setArguments(args);
    		return p;
    	}
    
    	/*
    	 * (non-Javadoc)
    	 * 
    	 * @see android.support.v4.app.DialogFragment#onCreate(android.os.Bundle)
    	 */
    	@Override
    	public void onCreate(Bundle savedInstanceState) {
    		currentPage = getArguments().getInt("currentPage");
    		maxPages = getArguments().getInt("maxPages");
    		super.onCreate(savedInstanceState);
    	}
    
    	@Override
    	public Dialog onCreateDialog(Bundle savedInstanceState) {
    		LayoutInflater inflater = LayoutInflater.from(getActivity());
    		final View v = inflater.inflate(R.layout.page_dialog, null);
    		return new AlertDialog.Builder(getActivity())
    				.setTitle("Go to page...")
    				.setView(v)
    				.setCancelable(true)
    				.setPositiveButton("Ok!", new DialogInterface.OnClickListener() {
    					@Override
    					public void onClick(DialogInterface dialog, int which) {
    						//validation code
    					}
    				})
    				.setNegativeButton("Aww, hell no!", new DialogInterface.OnClickListener() {
					@Override
					public void onClick(DialogInterface dialog, int which) {
						dialog.cancel();
					}
				}).create();
    	}
    }
    
The important lines in the code snippet displayed above are 28, 29 and 32.  On 28, the app gets the inflater being used by the Activity].  Remember, you should never instantiate a :java:`LayoutInflater` directly.  After that, it's pretty trivial to generate the dialog's content (29) and then insert it into an :java:`AlertDialog`'s View hierarchy.

Run it and what do you get?

.. figure:: http://i.imgur.com/kIcj464l.png
    :alt: Honeycomb+ button order 
    :target: http://imgur.com/kIcj464
    
    Beautiful!
    
Now, run it in Froyo to see if the buttons are automagically reordered…
  
.. figure:: http://i.imgur.com/TjnGC9jl.png
    :alt: Gingerbread and down button order
    :target: http://imgur.com/TjnGC9j

    And there was much rejoicing.
    
*Disclaimer: Yes, I know I'm inlining strings that would be best stored in an external file. That's a step I put off until I'm ready to make a release build.*