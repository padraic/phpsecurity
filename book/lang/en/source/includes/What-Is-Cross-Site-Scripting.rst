What is Cross-Site Scripting?
=============================
 
XSS occurs when an attacker is capable of injecting a script, often Javascript, into the output of a web application in such a way that it is executed in the client browser. This ordinarily happens by locating a means of breaking out of a data context in HTML into a scripting context - usually by injecting new HTML, Javascript strings or CSS markup. HTML has no shortage of locations where executable Javascript can be injected and browsers have even managed to add more. The injection is sent to the web application via any means of input such as HTTP parameters.
 
One of the major underlying symptoms of Cross-Site Scripting's prevelance, unique to such a serious class of security vulnerabilities, is that programmers continually underestimate its potential for damage and commonly implement defenses founded on misinformation and poor practices. This is particularly true of PHP where poor information has overshadowed all other attempts to educate programmers. In addition, because XSS examples in the wild are of the simple variety programmers are not beyond justifying a lack of defenses when it suits them. In this environment, it's not hard to see why a 65% vulnerability rate exists.
 
If an attacker can inject Javascript into a web application's output and have it executed, it allows the attacker to execute any conceivable Javascript in a user's browser. This gives them complete control of the user experience. From the browser's perspective, the script originated from the web application so it is automatically treated as a trusted resource.
 
Back in my Introduction, I noted that trusting any data not created explicitly by PHP in the current request should be considered untrusted. This sentiment extends to the browser which sits separately from your web application. The fact that the browser trusts everything it receives from the server is itself one of the root problems in Cross-Site Scripting. Fortunately, it's a problem with an evolving solution which we'll discuss later.
 
We can extend this even further to the Javascript environment a web application introduces within the browser. Client side Javascript can range from the very simple to the extremely complex, often becoming client side applications in their own right. These client side applications must be secured like any application, distrusting data received from remote sources (including the server-hosted web application itself), applying input validation, and ensuring output to the DOM is correctly escaped or sanitised.
 
Injected Javascript can be used to accomplish quite a lot: stealing cookie and session information, performing HTTP requests with the user's session, redirecting users to hostile websites, accessing and manipulating client-side persistent storage, performing complex calculations and returning results to an attacker's server, attacking the browser or installing malware, leveraging control of the user interface via the DOM to perform a UI Redress (aka Clickjacking) attack, rewriting or manipulating in-browser applications, attacking browser extensions, and the list goes on...possibly forever.
 
UI Redress (also Clickjacking)
------------------------------
 
While a distinct attack in its own right, UI Redress is tightly linked with Cross-Site Scripting since both leverage similar sets of vectors. Sometimes it can be very hard to differentiate the two because each can assist in being successful with the other.
 
A UI Redress attack is any attempt by an attacker to alter the User Interface of a web application. Changing the UI that a user interacts with can allow an attacker to inject new links, new HTML sections, to resize/hide/overlay interface elements, and so on. When such attacks are intended to trick a user into clicking on an injected button or link it is usually referred to as Clickjacking.
 
While much of this chapter applies to UI Redress attacks performed via XSS, there are other methods of performing a UI Redress attack which use frames instead. I'll cover UI Redressing in more detail in Chapter 4.