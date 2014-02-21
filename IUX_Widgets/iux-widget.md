## One Intuit Identity Widgets
learnings & technical design



## Highly re-usable widgets

Requirements:

* 3 lines of code to get started
* 5 minutes or less onboarding
* configurable and customizable



## Convention over configuration

<pre><code data-trim style="word-wrap: break-word;">
    <div id="ius-sign-in-widget"></div>​
</code></pre>

<br />

* Placeholder to render HTML
* Name of the widget as the ID with prefix and suffix


<pre><code data-trim style="word-wrap: break-word;">
&lt;script
    src="https://accounts-e2e.intuit.com​/IUS-Plugins/v2/scripts/ius.min.js"
    data-widgets="sign-in,sign-up"&gt;
&lt;/script&gt;
</code></pre>

<br />

* Which environment it's pointing to
* Which version of widget JS
* Debug mode vs. minified mode
* Widgets that needs to be loaded


<pre><code data-trim style="word-wrap: break-word;">
&lt;script&gt;
    intuit.ius.signIn.setup({
        offeringId: 'Intuit.cto.iam.ius',
        enableOAuthFlow: true,
        onSignInSuccess: function() {
            window.location = 'https://www.intuit.com';
        }
    });
&lt;/script&gt;
</code></pre>

<br />

* Call setup to initialize the widget with configurable options


Meta tags

    <meta name="ius-theme" content="harmony">

    <meta name="ius-locale" content="fr_CA">

    <meta name="ius-script" content="sign-in,sign-up">

<br />

For script loaders libraries that does not allow

`data-attribute`

on the script tag, meta tag is used to configure the widget
Note: cross cutting concerns for convention, exceptions and corner cases for configuration



## Be a good tenant

Widget HTML/CSS/JavaScript lives and executes in an environment that we do not control, so ..


We load only __two__ core 3rd party libraries

 * jQuery (selector, ajax, events)
 * RequireJS - JS dependency management
 * RequireJS - i18n localization efforts


* Prefix Classes with "ius-" (prevent style bleeding out)
* CSS reset on the .ius class (prevent style bleeding in)
* Prefix Element IDs with "ius-" (prevent ID clashing)


Scope carefully and namespace everything

    (function(document, window, undefined) {
       if (typeof intuit === 'undefined' || !intuit) { intuit = {}; }
       if (!intuit.ius) { intuit.ius = {}; }
       . . .
    })(document, window);

<small>
Do not assume anything, in the world of JavaScript,<br />
even __document__, __window__ and __undefined__ object might not be what you expect
</small>


Scoping jQuery

<pre><code data-trim style="word-wrap: break-word;">
if(window.jQuery === undefined) {
	intuit.ius.require(files, function(){
		jQueryLoaded(false);
    });
}
// if host page jQuery is lower version than us we load 1.9.1
else if (versionCompare(window.jQuery.fn.jquery, jQueryVersion) == -1) {
	var olderVersion = window.jQuery;
	intuit.ius.require(files, function() {
		jQueryLoaded(true);
		window.jQuery = olderVersion;
	});
} else {
	jQueryLoaded(false);
}
</code></pre>


<pre><code data-trim style="word-wrap: break-word;">
function jQueryLoaded(sandbox) {
	if(sandbox) {
		intuit.ius.jQuery = jQuery.noConflict(true);
	} else {
		intuit.ius.jQuery = window.jQuery;
	}
}

// then every enclosed scope first thing we do is
var $ = intuit.ius.jQuery
</code></pre>


Sometimes it's not as elegant, but you have to do it.

<small>We went into requireJS source code, replaced

`window.require`

with

`intuit.ius.require`

and re-minify it

</small>



## Cross Domain Request - Now

How do [turbotax.intuit.com](https://turbotax.intuit.com) make request to
[accounts.platform.intuit.com](https://accounts.platform.intuit.com)?

Initial requirements:

1. Support browsers all the way back to IE 6/7
2. Needs to be _secure_ (authentication and authorization)

<br />

Decision: cannot use JSONP and CORS

<small>no libraries that relies on messaging using Flash or Silverlight</small>


Implementation borrowed from IPP

* iframe communication between parent and child frame by setting document.domain to __intuit.com__ on both parent and child frame
* iframe injection by ius-apis.js which stands on its own. (ius.js loads this file as dependency)
* turbotax.intuit.com talks to accounts.intuit.com through frames
* accounts.intuit.com server proxy call to accounts.platform.intuit.com


However there are many down sides to this method.

* Page redirection for async calls initiates from the child iframe so it loses context
* Modifies document.domain which is a global variable that should NOT be touched
* Due to document.domain manipulation it does not play well with modern MVC framework such as Angular in IE (navigation)



## Cross Domain Request - Future

* Browser feature detection for CORS and falls back to iframe
* Enable CORS either on accounts.intuit.com's proxy or accounts.platform.intuit.com via Gateway



## Customization - JavaScript

PD teams will want to customize everything; sometimes for good reasons since they know their customers the best.

<br />

PD teams are also interested in metrics.


For example

    intuit.ius.signUp.setup({
        . . .
        signInLink : function() { alert('win'); },
        accountRecoveryLink: '/account-recovery.html',
        showName: true,
        isLowSecurity: false,
        // success call back handler
        onSignUpSuccess: function SignUpSuccess(responseObj) {},
        // fail call back handler
        onSignUpFail: function SignUpFail(errorJsonObj) {},
        onLoad: function SignUpSuccess(response) { console.log('loaded!'); }
    });

* Links on the widget can either take in a URL or an onClick function handler
* Every success/fail server call event should be propagated back via call back handlers, even if its just for analytics sakes
* Functional customization such as ability to create low security account, or create account with first and last name should be available


Think about boundaries and responsibility between the widget and the host

* At which point(s) do you give back control to the host?
* Auto focus into widget's input field for good usability? Probably not if the host page auto focus into a search bar.


## Customization - CSS

We provide themes out of the box for the widget

1. HARMONY theme (SBG)
2. CONSUMER theme (TurboTax)
3. DEFAULT theme (heavily influenced by Harmony)


CSS architecture

* We have ius-base-theme.css and ius-base-layout.css.
    * Theme is for look and feel and we support overriding styles in theme files
    * Layout is for positioning the elements. We do not support overriding layout styles. (PD team can still override it but we do not guarantee backward compatibility)
* Then we have individual widget theme and layout CSS files
* Also mobile theme and layouts
* Soon we will also have locale specific theme and layout


Manual override CSS theme

    <style id="ius-theme-override" src=".." />

for the overriding CSS, add an ID "ius-theme-override" which will tell IUS to inject CSS style before this tag for the cascade rules to apply.


## Customization - Content

We also support overriding the label/content of the the widgets. As well as errorMessages shown to the user.

    intuit.ius.signUp.setup({
        ...
        contentOverride : [
			{id: "ius-sign-up-header", content: "Créer un compte"},
			{id: "ius-sign-in-url", content: "Ouvrir une session"},
			...],

		errorMessageOverride = [
		    {responseCode: "DUPLICATE_USER", message:"Vous avez déjà un compte avec Intuit."},
			// if it's id then it implies it's input validation
			{id: "ius-email", message: "Veuillez entrer un courriel valide."},
			...]
		...
    });

* Give us the ID of the element, we will check the element type and override the appropriate attribute with the content
* Or the corresponding errorCode and we will override the default error message



## Widget JavaScript Boilerplate

    intuit.ius.define(['i18n!i18n/nls/tokens',
	    intuit.ius.enableMinification ? 'ius-idproofing.min' : 'ius-idproofing' ...
	    . . .],  function(tokens) {
	    intuit.ius.accountRecovery = function(tokens) {

            // view
            var renderer = function() { . . . }();

            // model
            var data = function() { . . . }();

            // controller
            var events = function() { . . . }();

            // helpers
            var apis = function() { .. }();      // help invoke backend API
            var analytics = function() { .. }(); // beacon back to omniture
            var toErrorMessage = function() { .. }(); // error code to message mapping
            var selectElements = function() { .. }; // select HTML elements so we work with jQuery objects
            var validator = function() { .. }(); // input validation

            // fetching HTML via JSONP and CSS for widget
            var init = function() {
                if(intuit.ius.enableMinification) {
                    intuit.ius.loader.loadCss(minCssUrl);
              	} else {

              	for(var i = 0, j = debugCssUrls.length; i < j; i++) {
              		intuit.ius.loader.loadCss(debugCssUrls[i]);
              	}

                intuit.ius.loader.loadHtml(htmlUrl, function(html) {
                    widgetEl = $('#ius-account-recovery-widget');
                    events.onWidgetLoad(html);
                });
            }

            var retObj = {
            	    setup: function(o) {

            			// configuration for the widget
            			config = {
            				offeringId : o.offeringId,
            				// default values for all options
            				startWithPii : o.startWithPii || false,
            				. . .
            			};

                        // once config is set we can initialized
            			init();

                        // expose config option and init for console testing
            			var debugFlag = intuit.ius.urlParams['debug'] === 'true';
            			if(debugFlag) {
            				retObj.config = config;
            				retObj.init = init;
            			}
            		}
            	};

            	return retObj;

	    }(tokens);
	});



## Widget HTML Boilerplate

    <div id="ius-sign-in-wrapper" class="ius-reset ius ius-sign-in-widget">
        <section id="ius-sign-in" class="ius-hide" >
            <header class="ius-default-bottom-spacer">
                <span id="ius-sign-in-header" class="ius-header"><spring:message code="sign.in.header"/></span>
                <span id="ius-sign-in-sub-header" class="ius-sub-header"></span>
            </header>

            <div id="ius-sign-in-messages">
                <div id="ius-sign-in-error" aria-live="assertive" role="alert" class="ius-hide ius-default-bottom-margin ius-error-alert"></div>
            </div>

            <form action="#" method="post" autocomplete="on" novalidate>
                <fieldset  id="ius-fieldset-userid">
                    <label id="ius-label-userid" class="ius-label" for="ius-userid">
                        <spring:message code="sign.in.email"/>
                        <span class="ius-sub-label"><spring:message code="sign.in.user.id"/></span>
                    </label>
                    <div class="ius-text-input-container">
                        <input id="ius-userid" class="ius-text-input" name="Email" type="email" aria-required="true" required="required" data-valid="valid" />
                        <span  id="ius-status-userid" class="ius-status" aria-live="assertive" role="alert"></span>
                    </div>
                    <span  id="ius-error-userid" class="ius-error ius-field-block" for="ius-userid" aria-live="assertive" role="alert" ></span>
                </fieldset>

                <fieldset  id="ius-fieldset-password">
                    <label id="ius-label-password" class="ius-label" for="ius-sign-in-password"><spring:message code="password"/></label>
                    <div class="ius-text-input-container">
                        <input id="ius-password" class="ius-text-input" name="Password" type="password" aria-required="true" required="required" data-valid="valid" />
                        <span  id="ius-status-password" class="ius-status" aria-live="assertive" role="alert"></span>
                    </div>
                    <span  id="ius-error-password" class="ius-error ius-field-block" for="ius-password" aria-live="assertive" role="alert" ></span>
                </fieldset>

                . . .

                <div  id="ius-sign-in-actions" class="ius-default-bottom-spacer ius-clearfix" >
                    <input id="ius-sign-in-submit-btn" class="ius-btn ius-btn-submit ius-display-inline-block ius-float-left" name="SignIn" type="submit"
                        value="<spring:message code="sign.in" />" data-clicked-text="<spring:message code="signing.in" />" />
               </div>
            </form>
        </section>

        <section id="ius-userid-verification" class="ius-hide" >
            . . .
        </section>
    </div>

Avoid long flow with 6+ screens/views because it makes navigation back and forth troublesome.



## Client side error handling and logging

![browsers](images/browsers-logo.jpeg)

+

![plugins](images/browser-plugins.jpg)

+

Mobile OS, 56K dial up, firewall, antivirus .. (on and on)

=

Really?  . . . Yes


* Add an event to window.onError and send the error back to the server.
* However the host could be using this at the global scope
* It's hard to filter out which error is coming from widget.
* We are currently doing encompassing try/catch blocks.
* Try/Catch: the good - we have the stack traces. The bad - Async call back exceptions will not be caught (and ugly code).
* Use JSONP or GET fake image to record your JS error



## Challenges

* Connectivity and spotty connection will affect how your script behaves, gracefully handling it is key and it's a hard thing to do.
* Slow browsers will also affect how your script behaves and loads, race condition can happen and happen quite a bit particularly in IE. Be aware of script loaders and when stuff comes into scope.
* How to reduce amount of GET requests back to the server for speedier load time
* Analytics is hard since usually we don't have an end to end view. Hopefully Intuit Analytics Cloud will help.




## Other topics

* Localization framework
* Native app integration with OAuth2
* Caching with CDN and busting browser cache
* Share Modules within widgets (Widgets within widgets!)



## Thanks

IUSTeam@intuit.com