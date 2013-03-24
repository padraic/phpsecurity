Insufficient Entropy For Random Values
######################################

Random values are everywhere in PHP. They are used in all frameworks, many libraries and you probably have tons of code relying on them for generating tokens, salts, and as inputs into further functions. Random values are important for a wide variety of use cases.

1. To randomly select options from a pool or range of known options.
2. To generate initialisation vectors for encryption.
3. To generate unguessable tokens or nonces for authorisation purposes.
4. To generate unique identifiers like Session IDs.

All of these have a specific weakness. If any attacker can guess or predict the output from the Random Number Generator (RNG) or Pseudo-Random Number Generator (PRNG) you use, they will be able to correctly guess the tokens, salts, nonces and cryptographic initialisation vectors created using that generator. Generating high quality, i.e. extremely difficult to guess, random values is important. Allowing password reset tokens, CSRF tokens, API keys, nonces and authorisation tokens to be predictable is not the best of ideas!

The two potential vulnerabilities linked to random values in PHP are:

A. Information Disclosure
B. Insufficient Entropy

Information Disclosure, in this context, refers to the leaking of the internal state, or seed value, of a PRNG. Leaks of this kind can make predicting future output from the PRNG in use much easier. Insufficient Entropy refers to the initial internal state or seed of a PRNG being so limited that it or the PRNG's actual output is restricted to a more easily brute forcible range of possible values. Neither is good news for PHP programmers.

We'll examine both in greater detail with a practical attack scenario outlined soon but let's first look at what a random value actually means when programming in PHP.

What Makes A Random Value?
==========================
 
Confusion over the purpose of random values is further muddled by a common misconception. You have undoubtedly heard of the difference between cryptographically strong random values versus nebulous "all other uses" or "unique" values. The prevailing impression is that only those random values used in cryptography require high quality randomness (or, more correctly, high entropy) but all other uses can squeek by with something less. I'd argue that the above impression is false and even counterproductive. The true division is between random values that must never be predictable and those which are used for wholly trivial purposes where predicting them can have no harmful effect. This removes cryptography from the question altogether. In other words, if you are using a random value for a non-trivial purpose, you should automatically gravitate towards using much stronger RNGs.
 
The factor that makes a random value strong is the entropy used to generate it. Entropy is simply a measure of uncertainty in "bits". For example, if I take any binary bit, it can have a value of either 0 or 1. If an attacker has no idea which it is, we have an entropy of 2 bits (i.e. it's a coin toss with 2 possible outcomes). If an attacker knows it will always be 1, we have an entropy of 0 bits because predictability is the opposite of uncertainty. You can also have bit values between 0 and 2 if it's not a fair coin toss. For example, if the binary bit is 1 99% of the time, the entropy can only be a fraction above 0 bits. So, in general, the more uncertain binary bits we use, the better.

We can see this more clearly close to home in PHP. The mt_rand() function generates random values which are always digits. It doesn't output letters, special characters, or any other byte value. This means that an attacker needs far fewer guesses per byte, i.e. its entropy is low. If we substituted mt_rand() by reading bytes from the Linux /dev/random source, we'd get truly random bytes fed by environmental noise from the local system's device drivers and other sources. This second option is obviously much better and would provide substantially more bits of entropy.

The other black mark against something like mt_rand() is that it is not a true random generator. It is a Pseudorandom Number Generator (PRNG) or Deterministic Random Bit Generator (DRBG). It implements an algorithm called Mersenne Twister (MT) which generates numbers distributed in such a way as to approximate truly random numbers. It actually only uses one random value, known as the seed, which is then used by a fixed algorithm to generate other pseudorandom values.

Have a look at the following example which you can test locally.

.. code-block:: php

    mt_srand(1361152757.2);

    for ($i=1; $i < 25; $i++) {
        echo mt_rand(), PHP_EOL;
    }

The above script is a simple loop executed after we've seeded PHP's Marsenne-Twister function with a predetermined value (using the output from the example function in the docs for mt_srand() which used the current seconds and microseconds). If you execute this script, it will print out 25 pseudorandom numbers. They all look random, there are no collisions and all seems fine. Run the script again. Notice anything? Yes, the next run will print out the EXACT SAME numbers. So will the third, fourth and fifth run. This is not always a guaranteed outcome given variations between PHP versions in the past but this is irrelevant to the problem since it does hold true in all modern PHP versions.

If the attacker can obtain the seed value used in PHP's Mersenne Twister PRNG, they can predict all of the output from mt_rand(). When it comes to PRNGs, protecting the seed is paramount. If you lose it, you are no longer generating random values... This seed can be generated in one of two ways. You can use the mt_srand() function to manually set it or you can omit mt_srand() and let PHP generate it automatically. The second is much preferred but legacy applications, even today, often inherit the use of mt_srand() even if ported to higher PHP versions.

This raises a risk whereby the recovery of a seed value by an attacker (i.e. a successful Seed Recovery Attack) provides them with sufficient information to predict future values. As a result, any application which leaks such a seed to potential attackers has fallen afoul of an Information Disclosure vulnerability. This is actually a real vulnerability despite its apparently passive nature. Leaking information about the local system can assist an attacker in follow up attacks which would be a violation of the Defense In Depth principle.

Random Values In PHP
====================

PHP uses three PRNGs throughout the language and both produce predictable output if an attacker can get hold of the random value used as the seed in their algorithms.

1. Linear Congruential Generator (LCG), e.g. lcg_value()
2. The Marsenne-Twister algorithm, e.g. mt_rand()
3. Locally supported C function, i.e. rand()

The above are also reused internally for functions like array_rand() and uniqid(). You should read that as meaning that an attacker can predict the output of these and similar functions leveraging PHP's internal PRNGs if they can recover all of the necessary seed values. It also means that multiple calls to PRNGs do not convey additional protection beyond obscuration (and nothing at all in open source applications where the source code is public knowledge). An attacker can predict ALL outcomes for any known seed value.

In order to generate higher quality random values for use in non-trivial tasks, PHP requires external sources of entropy supplied via the operating system. The common option under Linux is /dev/urandom which can be read directly or accessed indirectly using the openssl_pseudo_random_bytes() or mcrypt_create_iv() functions. These two functions can also use a Windows cryptographically secure pseudorandom generator (CSPRNG) but PHP currently has no direct userland accessor to this without the extensions providing these functions. In other words, make sure your servers' PHP version has the OpenSSL or Mcrypt extensions enabled.

The /dev/urandom source is itself a PRNG but it is frequently reseeded from the high entropy /dev/random resource which makes it impractical for an attacker to target. We try to avoid directly reading from /dev/random because it is a blocking resource, if it runs out of entropy all reads will be blocked until sufficient entropy has been captured from the system environment. You should revert to /dev/random, obviously, for the most critical of needs when necessary.

All of this leads us to the following rule...

::

    All processes which require non-trivial random numbers MUST attempt to use
    openssl_pseudo_random_bytes(). You MAY fallback to mcrypt_create_iv() with
    the source set to MCRYPT_DEV_URANDOM. You MAY also attempt to directly read
    bytes from /dev/urandom. If all else fails, and you have no other choice,
    you MUST instead generate a value by strongly mixing multiple sources of
    available random or secret values.

You can find a reference implementation of this rule in `the SecurityMultiTool reference library`_. As is typical PHP Internals prefers to complicate programmer's lives rather than include something secure directly in PHP's core.

.. _the SecurityMultiTool reference library: https://github.com/padraic/SecurityMultiTool/blob/master/library/SecurityMultiTool/Random/Generator.php

Enough theory, let's actually look into how we can attack an application with this information.

Attacking PHP's Random Number Generators
========================================

In practice, PHP's PRNGs are commonly used in non-trivial tasks for various reasons.

The openssl_pseudo_random_bytes() function was only available in PHP 5.3 and had blocking problems in Windows until 5.3.4. PHP 5.3 also marked the time from which the MCRYPT_DEV_URANDOM source was supported for Windows in the mcrypt_create_iv() function. Prior to this, Windows only supported MCRYPT_RAND which is effectively the same system PRNG used internally by the rand() function. As you can see, there were a lot of coverage gaps prior to PHP 5.3 so a lot of legacy applications written to earlier PHP versions may not have switched to using stronger PRNGs.

The Openssl and Mcrypt extensions are also optional. Since you can't always rely on their availability even on servers with PHP 5.3 installed, applications will often use PHP's PRNGs as a fallback method for generating non-trivial random values.

In both of these scenarios, we have non-trivial tasks relying on random values generated using PRNGs seeded with low entropy values. This leaves them vulnerable to Seed Recovery Attacks. Let's take a simple example and actually demonstrate a realistic attack.

Imagine that we have located an application online which uses the following source code to generate tokens throughout the application for a variety of purposes.

.. code-block:: php

    $token = hash('sha512', mt_rand());
 
There are certainly more complicated means of generating a token but this is a nice variant with only one call to mt_rand() that is hashed using SHA512. In practice, if a programmer assumes that PHP's random value functions are "sufficiently random", they are far more likely to utilise a simple usage pattern so long as it doesn't involve the "cryptography" word. Non-cryptographic uses may include access tokens, CSRF tokens, API nonces and password reset tokens to name a few. Let me describe the characteristics of this vulnerable application in greater detail before we continue any further so we have some insight into the factors making this application vulnerable.

Vulnerable Application Characteristics
--------------------------------------

This isn't an exhaustive list - vulnerable characteristics can vary from this recipe!

1. The server uses mod_php allowing multiple requests to be served by the same PHP process when using KeepAlive
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

This is important for a simple reason - PHP's random number generators are seeded only once per process. If we can send 2+ requests to the same PHP process, that process will reuse the same seed. The whole point of the attack I'll be describing is to use one token disclosure to derive the seed and employ that to guess another token generated from the SAME seed (i.e. in the same process). While mod_php is ideal where multiple requests are necessary to gather related random values, there are certainly cases where several mt_rand() based values can be obtained using just one request. This would make any requirement for mod_php redundant. For example, some of the entropy used to generate the seed for mt_rand() may also be leaked through Session IDs or through values output in the same request.

2. The server exposes CSRF, password reset, or account confirmation tokens generated using mt_rand() based tokens
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
 
In order to derive a seed value, we want to be able to directly inspect a number generated by PHP's random number generators. The usage of this number doesn't actually matter so we can source this from any value we can access whether it be a naked mt_rand() output or a hashed CSRF or account confirmation token on signup. There may even be indirect sources where the random value determines other behaviour in output which gives the original value away. The main limitation is that it must be from the same process which generates a second token we're trying to predict. For those keeping the introduction in mind, this is an Information Disclosure vulnerability. As we'll soon see, leaking the output from PHP's PRNGs can be extremely dangerous. Note that this vulnerability is not limited to a single application - you can read PRNG output from one application on the server to determine output from another application on that server so long as the same PHP process is used for both.
 
3. Known weak token generation algorithm
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
 
You can figure this out by targeting an open source application, bribing an employee with access to private source code, finding a particularly peeved off former employee, or by guessing. Some token generating methods are more obvious than others or simply more popular. A truly weak means of generation will feature the use of one of PHP's random number generators (e.g. mt_rand()), weak entropy (no other source of uncertain data), and/or weak hashing (e.g. MD5 or no hashing whatsoever). The example code we're using generates tokens with some of these factors in evidence. I also included SHA512 hashing to demonstrate that obscuration is simply never a solution. SHA512 is actually a weak hashing solution in the sense that it is fast to compute, i.e. it allows an attacker to brute force inputs on any CPU or GPU at some incredible rates bearing in mind that Moore's Law ensures that that rate increases with each new CPU/GPU generation. This is why passwords must be hashed with something that requires a fixed time to execute irrespective of CPU/GPU performance or Moore's Law.

Executing The Attack
--------------------

Our attack is going to fairly simple. We're going to send two separate HTTP requests in rapid succession across a connection to a PHP process that the server will keep alive for the second request. We'll call them Request A and Request B. Request A targets an accessible token such as a CSRF token, a password reset token (sent to attacker via email) or something of similar nature (not forgetting other options like inline markup, arbitrary IDs used in queries, etc.). This initial token is going to be tortured until it surrenders its seed value. This part of the execution is a Seed Recovery Attack which relies on the seed having so little entropy that it can be brute forced or looked up in a pre-computed rainbow table.

Request B targets something far more interesting. Let's send a request to reset the local Administrator's account password. This will trigger some logic where a token is generated (using a random number based on the same seed as Request A if we fit both requests successfully onto the same PHP process). That token will be stored to the database in anticipation of the Administrator using a password reset link sent to them by email. If we can extract the seed for Request A's token then, having knowledge of how Request B's token is generated, we may predict that password reset token (and hit the reset link before the Admin reads the email!).

Here's the sequence of events as they will unfold:

1. Use Request A to obtain a token which we will reverse engineer to discover the seed value.
2. Use Request B to have a token based on the same seed value stored to the application's database for a password reset.
3. Crack the SHA512 hash to get hold of the random number generated originally by the server.
4. Use the random value we cracked to brute force the seed value used to generate it.
5. Use the seed to generate a series of random values likely to have been the basis of the password reset token.
6. Use our password reset token(s) to reset the Administrator's password.
7. Gain access to the Administrator's account for fun and profit. Well, fun at least.
 
Let's get hacking...

Hacking The Application Step By Step
------------------------------------

Step 1: Carry out Request A to fetch a token of some description
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

We're operating on the basis that the target token and the password reset token both depend on the output from mt_rand() so we need to select this carefully. In our case, this imaginative scenario is an application where all tokens are generated the same way so we can just take a short trip to extract a CSRF token and store it somewhere for later reference.

Step 2: Carry out Request B to have a password reset token issued for the Administrator account
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

This request is a simple matter of submitting a password reset form. The token will be stored to the database and sent to the user in an email. This is the token we now have to calculate correctly. If the server's characteristics are accurate, this request will reuse the same PHP process as Request A thus ensuring that both calls to mt_rand() are using the same identical seed value. We could even just use Request A to grab the reset form's CSRF token to enable the submission to streamline things (cut out a middle round trip).

Step 3: Crack the SHA512 hashing on the token retrieved from Request A
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

SHA512 inspires awe in programmers because it's the biggest number available in the SHA-2 family of algorithms. However, the method our target is using to generate tokens suffers from one flaw - random values are restricted to digits (i.e. its uncertainty or entropy is close to negligible). If you check the output from mt_getrandmax(), you'll discover that the maximum random number mt_rand() can generate is only 2.147 billion with some loose change. This limited number of possibilities make it ripe for a brute force attack.

Don't take my word for it though. If you have a discrete GPU from the last few generations, here's how you get started. I opted to use the excellent hashcat-lite since I'm only looking at a single hash. This version of hashcat is one of the fastest such brute forcing tools and is available for all major operating systems including Windows. You can download it from http://hashcat.net/oclhashcat-lite/ in a few seconds.

Generate a token using the method I earlier prescribed using the following script:

.. code-block:: php

    $rand = mt_rand();
    echo "Random Number: ", $rand, PHP_EOL;
    $token = hash('sha512', $rand);
    echo "Token: ", $token, PHP_EOL;

This simulates the token from Request A (which is our SHA512 hash hiding the generated random number we need) and run it through hashcat using the following command.

::

    ./oclHashcat-lite64 -m1700 --pw-min=1 --pw-max=10 -1?d -o ./seed.txt <SHA512 Hash> ?d?d?d?d?d?d?d?d?d?d

Here's what all the various options mean:

+ -m1700: Specifies the hashing algo where 1700 means SHA512.
+ --pw-min=1: Specifies the minimum input length of the hashed value.
+ --pw-max=10: Specifies the maximum input length of the hashed value (10 for mt_rand()).
+ -1?d: Specifies that we want a custom dictionary of only digits (i.e. 0-9)
+ -o ./seed.txt: Output file where results will be written. None are printed to screen so don't forget it!
+ ?d?d?d?d?d?d?d?d?d?d: The mask showing the format to use (all digits to max of 10).

If all works correctly, and your GPU does not explode, Hashcat will figure out what random number was hashed in a couple of minutes. Yes, minutes. I spent some time earlier explaining how entropy works and here you can see it in practice. The mt_rand() function is limited to so few possibilities that the SHA512 hashes of all possible values can be computed in a very short time. The use of hashing to obscure the output from mt_rand() was basically useless.

Step 4: Recover the seed value from the newly cracked random number
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

As we saw above, cracking any mt_rand() value from its SHA512 hash only requires a couple of minutes. This should give you a preview of what happens next. With the random value in hand we can run another `brute forcing tool called php_mt_seed`_. This is a small utility that was written to take any output of mt_rand() and perform a brute force attack to locate a seed that would generate that value. You can download the current version, compile it, and run it as follows. You can use an older version if you have compile problems (newer versions had issues with virtual environments when I was testing).

.. _brute forcing tool called php_mt_seed: http://download.openwall.net/pub/projects/php_mt_seed/

::

    ./php_mt_seed <RANDOM NUMBER>

This might take a bit more time than cracking the SHA512 hash since it's CPU bound, but it will search the entire possible seed space inside of a few minutes on a decent CPU. The result will be one or more candidate seeds (i.e. seeds which produce the given random number). Once again, we're seeing the outcome of weak entropy, though this time as it pertains to how PHP generates seed values for its Marsenne-Twister function. We'll revisit how these seeds are generated later on so you can see why such a brute forcing attack is possible in such a spectacularly short time.

In the above steps, we made use of simple brute forcing tools that exist in the wild. Just because these tools have a narrow focus on single mt_rand() calls, bear in mind that they represent proofs of concept that can be modified for other scenarios (e.g. sequential mt_rand() calls when generating tokens). Also bear in mind that the cracking speed does not preclude the generation of rainbow tables tailored to specific token generating approaches. Here's another generic tool written in Python which targets PHP mt_rand() vulnerabilities: https://github.com/GeorgeArgyros/Snowflake

Step 5: Generate Candidate Password Reset Tokens for Administrator Account
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

Assuming that the total calls to mt_rand() across both Request A and Request B were just two, you can now start predicting the token with the candidate seeds using:

.. code-block:: php

    function predict($seed) {
        /**
         * Seed the PRNG
         */
        mt_srand($seed);
        /**
         * Skip the Request A call to the function
         */
        mt_rand();
        /**
         * Predict and return the Request B generated token
         */
        $token = hash('sha512', mt_rand());
        return $token;
    }

This function will predict the reset token for each candidate seed.

Step 6 and 7: Reset the Administator Account Password/Be naughty!
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

All you need to do now is construct a URL containing the token which will let you reset the Administrator's password via the vulnerable application, gain access to their account, and probably find out that they can post unfiltered HTML to a forum or article (another Defense In Depth violation that can be common). That would allow you to mount a widespread Cross-Site Scripting (XSS) attack on all other application users by infecting their PCs with malware and Man-In-The-Browser monitors. Seriously, why stop with just access? The whole point of these seemingly passive, minor and low severity vulnerabilities is to help attackers slowly worm their way into a position where they can achieve their ultimate goal. Hacking is like playing an arcade fighting game where you need combination attacks to pull off some devastating moves.

Post-Attack Analysis
--------------------

The above attack scenario, and the ease with which the varous steps are executed, should clearly demonstrate the dangers of mt_rand(). In fact, the risks are so clear that we can now consider any weakly obscured output of a mt_rand() value in any form accessible to an attacker as an Information Disclosure vulnerability.
 
Furthermore, there are two sides to the story. For example, if you depend on a library innocently using mt_rand() for some important purpose without ever outputting such values, your own separate use of a leaky token may compromise that library. This is problematic because the library, or framework, in question is doing nothing to mitigate against Seed Recovery Attacks. Do we blame the user for leaking mt_rand() values or the library for not using better randomness?
 
The answer to that is that there is enough blame to go around for both. The library should not be using mt_rand() (or any other single source of weak entropy) for any sensitive purposes as its sole source of random values, and the user should not be writing code that leaks mt_rand() values to the world. So yes, we can actually start pointing fingers at unwise uses of mt_rand() even where those uses are not directly leaking to attackers.

So not only do we have to worry about Information Disclosure vulnerabilities, we also need to be conscious of Insufficient Entropy vulnerabilities which leave applications vulnerable to brute force attacks on sensitive tokens, keys or nonces which, while not technically cryptography related, are still used for important non-trivial functions in an application.

And Now For Something Completely Similar
========================================

Knowing now that an application's use of PHP's PRNGs can be interpreted as Insufficient Entropy vulnerabilities (i.e. they make brute forcing attacks easier by reducing uncertainty), we can extend our targets a bit more to something we've likely all seen somewhere.

.. code-block:: php

    $token = hash('sha512', uniqid(mt_rand()));

Assuming the presence of an Information Disclosure vulnerability, we can now state that this method of generating tokens is completely useless also. To understand why this is so, we need to take a closer look at PHP's uniqid() function. The definition of this function is as follows:

Gets a prefixed unique identifier based on the current time in microseconds. 

If you remember from our discussion of entropy, you measure entropy by the amount of uncertainty it introduces. In the presence of an Information Disclosure vulnerability which leaks mt_rand() values, our use of mt_rand() as a prefix to a unique identifier has zero uncertainty. The only other input to uniqid() in the example is time. Time is definitely NOT uncertain. It progresses in a predictable linear manner. Predictable values have very low entropy.

Of course, the definition notes "microseconds", i.e. millionths of a second. That provides 1,000,000 possible numbers. I ignore the larger seconds value since that is so large grained and measurable (e.g. the HTTP Date header in a response) that it adds almost nothing of value. Before we get into more technical details, let's dissect the uniqid() function by looking at its C code.

.. code-block:: c

    gettimeofday((struct timeval *) &tv, (struct timezone *) NULL);
    sec = (int) tv.tv_sec;
    usec = (int) (tv.tv_usec % 0x100000);

    /* The max value usec can have is 0xF423F, so we use only five hex
     * digits for usecs.
     */
    if (more_entropy) {
        spprintf(&uniqid, 0, "%s%08x%05x%.8F", prefix, sec, usec, php_combined_lcg(TSRMLS_C) * 10);
    } else {
        spprintf(&uniqid, 0, "%s%08x%05x", prefix, sec, usec);
    }

    RETURN_STRING(uniqid, 0);

If that looks complicated, you can actually replicate all of this in plain old PHP:

.. code-block:: php

    function unique_id($prefix = '', $more_entropy = false) {
        list($usec, $sec) = explode(' ', microtime());
        $usec *= 1000000;
        if(true === $more_entropy) {
            return sprintf('%s%08x%05x%.8F', $prefix, $sec, $usec, lcg_value()*10);
        } else {
            return sprintf('%s%08x%05x', $prefix, $sec, $usec);
        }
    }
 
This code basically tells us that a simple uniqid() call with no parameters will return a string containing 13 characters. The first 8 characters are the current Unix timestamp (seconds) in hexadecimal. The final 5 characters represent any additional microseconds in hexadecimal. In other words, a basic uniqid() will provide a very accurate system time measurement which you can dissect from a simple uniqid() call using something like this:

.. code-block:: php

    $id = uniqid();
    $time = str_split($id, 8);
    $sec = hexdec('0x' . $time[0]);
    $usec = hexdec('0x' . $time[1]);
    echo 'Seconds: ', $sec, PHP_EOL, 'Microseconds: ', $usec, PHP_EOL;
 
Indeed, looking at the C code, this accurate system timestamp is never obscured in the output no matter what parameters you use.

.. code-block:: php

    echo uniqid(), PHP_EOL;                 // 514ee7f81c4b8
    echo uniqid('prefix-'), PHP_EOL;        // prefix-514ee7f81c746
    echo uniqid('prefix-', true), PHP_EOL;  // prefix-514ee7f81c8993.39593322

Brute Force Attacking Unique IDs
================================
 
If you think about this, it becomes clear that disclosing any naked uniqid() value to an attacker is another example of a potential Information Disclosure vulnerability. It leaks an insanely accurate system time that can be used to guess the inputs into subsequent calls to uniqid(). This helps solves any dilemna you face with predicting microseconds by narrowing 1,000,000 possibilities to a narrower range. While this leak is worthy of mention for later, technically it's not needed for our example. Let's look at the original uniqid() token example again.

.. code-block:: php

    $token = hash('sha512', uniqid(mt_rand()));
 
Taking the above example, we can see that by combining a Seed Recovery Attack against mt_rand() and leveraging an Information Disclosure from uniqid(), we can now make inroads in calculating a narrower-then-expected selection of SHA512 hashes that might be a password reset or other sensitive token. Heck, if you want to narrow the timestamp range without any naked uniqid() disclosure leaking system time, server responses will typically have a HTTP Date header to analyse for a server-accurate timestamp. Since this just leaves the remaining entropy as one million possible microsecond values, we can just brute force this in a few seconds!

.. code-block:: php

    <?php
    echo PHP_EOL;

    /**
     * Generate token to crack without leaking microtime
     */
    mt_srand(1361723136.7);
    $token = hash('sha512', uniqid(mt_rand()));

    /**
     * Now crack the Token without the benefit of microsecond measurement
     * but remember we get seconds from HTTP Date header and seed for
     * mt_rand() using earlier attack scenario ;)
     */
    $httpDateSeconds = time();
    $bruteForcedSeed = 1361723136.7;
    mt_srand($bruteForcedSeed);
    $prefix = mt_rand();

    /**
     * Increment HTTP Date by a few seconds to offset the possibility of
     * us crossing the second tick between uniqid() and time() calls.
     */
    for ($j=$httpDateSeconds; $j < $httpDateSeconds+2; $j++) { 
        for ($i=0; $i < 1000000; $i++) {
            /** Replicate uniqid() token generator in PHP */
            $guess = hash('sha512', sprintf('%s%8x%5x', $prefix, $j, $i));
            if ($token == $guess) {
                echo PHP_EOL, 'Actual Token: ', $token, PHP_EOL,
                    'Forced Token: ', $guess, PHP_EOL;
                exit(0);
            }
            if (($i % 20000) == 0) {
                echo '~';
            }
        }
    }

Adding More Entropy Will Save Us?
---------------------------------

There is, of course, the option of adding extra entropy to uniqid() by setting the second parameter of the function to TRUE:

.. code-block:: php

    $token = hash('sha512', uniqid(mt_rand(), true));

As the C code shows, this new source of entropy uses output from an internal php_combined_lcg() function. This function is actually exposed to userland through the lcg_value() function which I used in my PHP translation of the uniqid() function. It basically combines two values generated using two separately seeded Linear Congruential Generators (LCGs). Here is the code actually used to seed these two LCGs. Similar to mt_rand() seeding, the seeds are generated once per PHP process and then reused in all subsequent calls.

.. code-block:: c

    static void lcg_seed(TSRMLS_D) /* {{{ */
    {
        struct timeval tv;

        if (gettimeofday(&tv, NULL) == 0) {
            LCG(s1) = tv.tv_sec ^ (tv.tv_usec<<11);
        } else {
            LCG(s1) = 1;
        }
        #ifdef ZTS
        LCG(s2) = (long) tsrm_thread_id();
        #else
        LCG(s2) = (long) getpid();
        #endif

        /* Add entropy to s2 by calling gettimeofday() again */
        if (gettimeofday(&tv, NULL) == 0) {
            LCG(s2) ^= (tv.tv_usec<<11);
        }

        LCG(seeded) = 1;
    }

If you stare at this long enough and feel tempted to smash something into your monitor, I'd urge you to reconsider. Monitors are expensive.

The two seeds both use the gettimeofday() function in C to capture the current seconds since Unix Epoch (relative to the server clock) and microseconds. It's worth noting that both calls are fixed in the source code so the microsecond() count between both will be minimal so the uncertainty they add is not a lot. The second seed will also mix in the current process ID which, in most cases, will be a maximum number of 32,768 under Linux. You can, of course, manually set this as high as ~4 million by writing to /proc/sys/kernel/pid_max but this is very unlikely to reach that high.

The pattern emerging here is that the primary source of entropy used by these LCGs is microseconds. For example, remember our mt_rand() seed? Guess how that is calculated.

.. code-block:: c

    #ifdef PHP_WIN32
    #define GENERATE_SEED() (((long) (time(0) * GetCurrentProcessId())) ^ ((long) (1000000.0 * php_combined_lcg(TSRMLS_C))))
    #else
    #define GENERATE_SEED() (((long) (time(0) * getpid())) ^ ((long) (1000000.0 * php_combined_lcg(TSRMLS_C))))
    #endif

You'll notice that this means that all seeds used in PHP are interdependent and even mix together similar inputs multiple times. You can feasibly limit the range of initial microseconds as we previously discussed, using two requests where the first hits the transition between seconds (so microtime with be 0 plus exec time to next gettimeofday() C call), and even calculate the delta in microseconds between other gettimeofday() calls with access to the source code (PHP being open source is a leg up). Not to mention that brute forcing a mt_rand() seed gives you the final seed output to play with for offline verification.

The main problem here is, however, php_combined_lcg(). This is the underlying implementation of the userland lcg_value() function which is seeded once per PHP process and where knowledge of the seed makes its output predictable. If we can crack that particular nut, it's effectively game over.

There's An App For That...
--------------------------

I've spent much of this article trying to keep things practical, so better get back to that. Getting the two seeds used by php_combined_lcg() is not the easiest task because it's probably not going to be directly leaked (e.g. it's XOR'd into the seed for mt_rand()). The userland lcg_value() function is relatively unknown and programmers mostly rely on mt_rand() if they need to use a PHP PRNG. I don't want to preclude leaking the value of lcg_value() somewhere but it's just not a popular function. The two combined LCGs used also do not feature a seeding function (so you can't just go searching for mt_srand() calls to locate really bad seeding inherited from someone's legacy code). There is however one reliable output that does provide some direct output for brute forcing of the seeds - PHP session IDs.

.. code-block:: c

    spprintf(&buf, 0, "%.15s%ld%ld%0.8F", remote_addr ? remote_addr : "", tv.tv_sec,
    (long int)tv.tv_usec, php_combined_lcg(TSRMLS_C) * 10);

The above generates a pre-hash value for the Session ID using an IP address, timestamp, microseconds and...the output from php_combined_lcg(). Given a significant reduction in microtime possibilities (the above needs 1 for generating the ID and 2 within php_combined_lcg() which should have minimum changes between them) we can now perform a brute forcing attack. Well, maybe.

As you may recall from earlier, PHP now supports some newer session options such as session.entropy_file and session.entropy_length. The reason for this was to prevent brute forcing attacks on the session ID that would quickly (as in not take hours) reveal the two seeds to the twin LCGs combined by php_combined_lcg(). If you are running PHP 5.3 or less, you may not have those settings properly configured which would mean you have another useful Information Disclosure vulnerability exposed which will enable brute forcing of session IDs to get the LCG seeds.

There's a Windows app to figure out the LCG seeds in such cases to prove the point:
http://blog.ptsecurity.com/2012/08/not-so-random-numbers-take-two.html

More interestingly, knowledge of the LCG states feeds into how mt_rand() is seeded so this is another path to get around any lack of mt_rand() value leaks.

What does this mean for adding more entropy to uniqid() return values?

.. code-block:: php

    $token = hash('sha512', uniqid(mt_rand(), true));

The above is another example of a potential Insufficient Entropy vulnerability. You cannot rely on entropy which is being leaked from elsewhere (even if you are not responsible for the leaking!). With the Session ID information disclosure leak, an attacker can predict the extra entropy value that will be appended to the ID.

Once again, how do we assign blame? If Application X relies in uniqid() but the user or some other application on the same server leak internal state about PHP's LCGs, we need to mitigate at both ends. Users need to ensure that Session IDs use better entropy and third-party programmers need to be concious that their methods of generating random values lack sufficient entropy and switch to better alternatives (even where only weak entropy sources are possible!).

Hunting For Entropy
===================

By itself, PHP is incapable of generating strong entropy. It doesn't even have a basic API for exposing OS level PRNGs that are reliable strong sources. Instead, you need to rely on the optional existence of the openssl and mcrypt extensions. Both of these extensions offer functions which are significant improvements over their leaky, predictable, low-entropy cousins.

Unfortunately, because both of these extensions are optional, we have little choice but to rely on weak entropy sources in some circumstances as a last ditch fallback position. When this happens, we need to supplement the weak entropy of mt_rand() by including additional sources of uncertainty and mixing all of these together into a single pool from which we can extract pseudo-random bytes. This form of random generator which uses a strong entropy mixer has already been implemented in PHP by `Anthony Ferrara`_ in his `RandomLib library on Github`_. Effectively, this is what programmers should be doing where possible.

.. _Anthony Ferrara: http://blog.ircmaxell.com
.. _RandomLib library on Github: https://github.com/ircmaxell/RandomLib

The one thing you want to avoid is the temptation to obscure your weak entropy by using hashing and complex mathmatical conversions. These are all readily repeatable by an attacker once they know which seeds to start from. These may impose a minor barrier by increasing the necessary computations an attacker must complete when brute forcing, but always remember that low entropy means less uncertainty - less uncertainty means fewer possibilities need to be brute forced. The only realistic solution is to increase the pool of entropy you're using with whatever is at hand.

Anthony's RandomLib generates random bytes by mixing various entropy sources and localised information which an attacker would need to work hard to guess. For example, you can mix mt_rand(), uniqid() and lcg_value() output and go further by adding the PID, memory usage, another microtime measurement, a serialisation of $_ENV, posix_times(), etc. You can go even further since RandomLib is extensible. For example, you could throw in some microsecond deltas (i.e. measure how many microseconds some functions take to complete with pseudo-random input such as hash() calls).

.. code-block:: php

    /**
     * Generate a 32 byte random value. Can also use these other methods:
     *  - generateInt() to output integers up to PHP_INT_MAX
     *  - generateString() to map values to a specific character range
     */
    $factory = new \RandomLib\Factory;
    $generator = $factory->getMediumStrengthGenerator();
    $token = hash('sha512', $generator->generate(32));

Arguably, due to RandomLib's footprint and the ready availability of the OpenSSL and Mcrypt extensions, you can instead use RandomLib as a fallback proposition as used in `the SecurityMultiTool PRNG generator class`_.

.. _the SecurityMultiTool PRNG generator class: https://github.com/padraic/SecurityMultiTool/blob/master/library/SecurityMultiTool/Random/Generator.php