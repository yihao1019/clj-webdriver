# Selenium-WebDriver Support for Clojure [![Build Status](https://secure.travis-ci.org/semperos/clj-webdriver.png)](http://travis-ci.org/semperos/clj-webdriver)

This is a Clojure library for driving a web browser using Selenium-WebDriver as the backend.

 * [Project Wiki](https://github.com/semperos/clj-webdriver/wiki)
 * [Marginalia Documentation (v0.4.0)](http://semperos.github.com/clj-webdriver/uberdoc-latest.html)
 * [Google Group](https://groups.google.com/forum/#!forum/clj-webdriver)
 * [Issue Queue](https://github.com/semperos/clj-webdriver/issues)
 * [Selenium-WebDriver Javadoc](http://selenium.googlecode.com/svn/trunk/docs/api/java/index.html)

**Please join the Google group if you use this library.** I regularly post announcements about upcoming releases, and although I ensure all tests are passing and try to maintain good test coverage before releases, user testing is invaluable. Thank you!

## Usage

### Important ###

 * This library uses *Clojure 1.3.0*.
 * You *must* add the java.net Maven repository to your own `project.clj` when using this library (for example: `:repositories {"java-dot-net" "http://download.java.net/maven/2"}`). The JNA jars required by the latest Selenium-WebDriver release are only available there.
 
### Quickstart ###

Use/require the library in your code:

```clj
(use 'clj-webdriver.core)
```

Start up a browser:

```clj
(def b (start :firefox "https://github.com"))
```

Here's an example of logging into Github:

```clj
;; Start the browser and bind it to `b`
(def b (start :firefox "https://github.com"))

;; Click "Login" link
(-> b
    (find-it {:text "Login"})
    click)

;; Input username/email into the "Login or Email" field
(-> b
    (find-it {:class "text", :name "login"}) ; use multiple attributes
    (input-text "username"))

;; Input password into the "Password" field
(-> b
    (find-it {:xpath "//input[@id='password']"}) ; :xpath and :css options
    (input-text "password"))

;; Click the "Log in" button"
(-> b
    (find-it :input {:value #"(?i)log"}) ; use of regular expressions
    click)
```

Filling out the form can been accomplished more compactly using `clj-webdriver.form-helpers/quick-fill` as follows:

```clj
(require '[clj-webdriver.form-helpers :as form])

(form/quick-fill b [{{:class "text", :name "login"}     "username"}
                    {{:xpath "//input[@id='password']"} "password"}
                    {{:value #"(?i)log"}                click}])
```

If you plan to submit the form, you need to pass a third parameter of `true` to prevent `quick-fill` from trying to return the elements you act upon (since the page will reload, they will be lost in the Selenium-WebDriver cache).

### Finding Elements ###

The `find-it` and `find-them` functions accept a variety of queries and return one or a seq of all matched elements respectively. Below is a list of query formats these functions accept:

* **HTML Tag as keyword:** Pass in the name of an HTML tag as a keyword (`:div`, `:a`, `:span`, `:img`, etc.) `(find-it :a)` will find the first `<a>` tag on the page. There are also special keywords such as `:*` (match any tag), `:text` (match textfields and textareas), `:window` (to match an open browser window by title or url)
* **HTML Tag plus attributes:** Pass in the name of an HTML tag as a keyword plus some attributes to describe it. `(find-it :a {:class "external"})` will return the first `<a>` tag with a class of "external"
* **HTML attributes alone:** You don't have to pass in a tag. `(find-it {:class "external"})` will find the first element of any tag with class "external"
* **Multiple HTML attributes:** You can pass in as many attribute-value pairs as you like. `(find-it {:class "external", :text "Moustache"})` will find the first HTML element on the page with both a class of "external" and visible text of "Moustache"
* **Regular Expressions:** Instead of looking for an exact match, you can use Java-style regular expressions to find elements. `(find-it :a {:class #"exter"})` will find the first `<a>` tag with a class which matches the regular expression `#"exter"`. You can also use regexes in the final position of an ancestry-based query (see below).
* **Ancestry-based queries:** This library provides a pure-Clojure mechanism for finding an element based on parent elements. `(find-it [:div {:id "content"}, :a {:class "external"}])` will find the first `<a>` tag with a class of "external" that is located within the `<div>` with id "content". This is equivalent to the XPath `//div[@id='content']//a[@class='external']`. You can also include regular expressions in the final attribute-value map which you supply. (*Note: Due to issues of ambiguity and in order not to reinvent the wheel any further, applying regexes higher up the query is not supported and will cause an exception. In addition, none of the "semantic" tags such as `:button*`, `:radio`, `:checkbox`, `:textfield`, etc. that do not map directly to HTML tags are not supported. If you need more advanced querying, use XPath or CSS selectors directly.)* 
* **XPath and CSS Selectors:** You can use the `:xpath` and `:css` attributes to use such queries in place of simple HTML attributes. If you use one of these attributes, you shouldn't use any others, as they will be ignored (e.g. `{:xpath "//a", :class "external"}` will only utilize the xpath `//a`). `(find-it {:xpath "//a[@class='external']"})` will return the first `<a>` tag with a class of "external"

To demonstrate how to use arguments in different ways, consider the following example. If I wanted to find `<a href="/contact" id="contact-link" class="menu-item" name="contact">Contact Us</a>` in a page and click on it I could perform any of the following:

```clj
(-> b
    (find-it :a)    ; assuming its the first <a> on the page
    click)

(-> b
    (find-it {:id "contact-link"})    ; :id is unique, so only one is needed
    click)

(-> b
    (find-it {:class "menu-item", :name "contact"})    ; use multiple attributes
    click)

(-> b
    (find-it :a {:class "menu-item", :name "contact"})    ; specify tag
    click)

(-> b
    (find-it :a {:text "Contact Us"})    ; special :text attribute, uses XPath's
    click)                               ; text() function to find the element

(-> b
    (find-it :a {:class #"(?i)menu-"})  ; use Java-style regular
    click)                               ; expressions

(-> b
    (find-it [:div {:id "content"}, :a {:id "contact-link"}]) ; hierarchical/ancestry-based query
    click)                                                    ; equivalent to
                                                              ; //div[@id='content']//a[@id='contact-link']

(-> b
    (find-it [:div {:id "content"}, :a {}]) ; ancestry-based query, tag with
    click)                                  ; no attributes (empty map required)

(-> b
    (find-it {:xpath "//a[@id='contact-link']"})    ; XPath query
    click)

(-> b
    (find-it {:css "a#contact-link"})    ; CSS selector
    click)
```

So, to describe the general pattern of interacting with the page:

```clj
(-> browser-instance
    (find-it options)
    (do-something-with-the-element))
```

### Firefox Functionality

Support for Firefox currently exceeds that for all other browsers, most notably via support for customizable Firefox profiles. I've included support for several of these advanced featues in the `clj-webdriver.firefox` namespace. Here are a few examples (borrowed from [here][wd-ruby-bindings]:

```clj
(use 'clj-webdriver.core)
(require '[clj-webdriver.firefox :as ff])

(def b (new-driver :firefox
                   (doto (ff/new-profile)
                         ;; Enable Firebug
                         (ff/enable-extension "/path/to/extensions/firebug.xpi")))
                         
                         ;; Auto-download certain file types to a specific folder
                         (ff/set-preferences {:browser.download.dir "C:/Users/semperos/Desktop",
                                              :browser.download.folderList 2
                                              :browser.helperApps.neverAsk.saveToDisk "application/pdf"})))
```
                                  
### Grid Support ###

If you already have a Selenium-WebDriver Grid (2) setup in place, you can now leverage the functions in `clj-webdriver.grid` to run your tests via the Grid.

At this point, `clj-webdriver.grid` has two functions, `new-driver-on-grid` and `start-on-grid` which are Grid equivalents for the `clj-webdriver.core` functions named `new-driver` and `start` respectively. You simply replace your call to `start` or `new-driver` with `start-on-grid` or `new-driver-on-grid` and your tests will run on the Grid.

For more information about configuring your Grid hub and nodes, read [the Selenium-WebDriver wiki documentation on Grid 2](http://code.google.com/p/selenium/wiki/Grid2).

## Running Tests

The namespace `clj-webdriver.test.example-app.core` contains a [Ring][ring-github] app (routing by [Moustache][moustache-github]) that acts as the "control application" for this project's test suite.

Use `lein test` to run this library's test suite. Ensure port 5744 is free, or edit `test/clj_webdriver/test/core.clj` before running the tests.

*Note:* If you just want to run the example app that clj-webdriver uses for its testing purposes, do the following:

 * Open a terminal and run `lein repl` or `lein swank` at the root of this project
 * Evaluate `(use 'clj-webdriver.test.example-app.core 'ring.adapter.jetty)`
 * Evaluate `(defonce my-server (run-jetty #'routes {:port 5744, :join? false}))`, making sure to adjust the `test-port` in `test/clj_webdriver/test/core.clj` to whatever you use here.

## Acknowledgements

Credits to [mikitebeka/webdriver-clj][webdriver-orig] for the initial code for this project and many of the low-level wrappers around the Selenium-WebDriver API.

Many thanks to those who have contributed so far (in nick-alphabetical order):

 * [maxweber](https://github.com/maxweber) (Max Weber)
 * [RobLally](https://github.com/RobLally) (Rob Lally)
 * [ulsa](https://github.com/ulsa) (Ulrik Sandberg)
 * [xeqi](https://github.com/xeqi) (Nelson Morris)

See Github for an [up-to-date list of contributors](https://github.com/semperos/clj-webdriver/contributors)

## License

Distributed under the [Eclipse Public License](http://opensource.org/licenses/eclipse-1.0.php), the same as Clojure.

[webdriver-orig]: https://github.com/mikitebeka/webdriver-clj
[ring-github]: https://github.com/mmcgrana/ring
[moustache-github]: https://github.com/cgrand/moustache
[wd-ruby-bindings]: http://code.google.com/p/selenium/wiki/RubyBindings
