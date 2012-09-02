Transport Layer Security (HTTPS, TLS and SSL)
#############################################

Communication between parties over the internet is fraught with risk. When you are sending payment instructions to your bank using their online facility, the very last thing you ever want to occur is for an attacker to be capable of intercepting, reading, manipulating or replaying the HTTP request to the online application. You can imagine the consequences of an attacker being able to read your session cookie, or to manipulate the payee targeted, or simply to inject new HTML or Javascript into the markup sent in response to a user request to the bank.

Protecting sensitive or private data is serious business. Application and browser users have an extremely high expectation in this regard placing a high value on the integrity of their credit card transactions, their privacy and their identity information. The answer to these concerns when it comes to defending the transfer of data from between any two parties is to use Transport Layer Security, typically involving HTTPS, TLS and SSL.

The goals of these security measures is as follows:

* To encrypt data being exchanged
* To guarantee the identity of one or both parties
* To prevent data tampering
* To prevent replay attacks

The most important point to notice in the above is that all four goals must be met in order for Transport Layer Security to be successful. If any one of the above are compromised, we have a real problem.

A common misconception, for example, is that encryption is the core goal and the others are non-essential. This is, in fact, completely untrue. Encryption of the data being transmitted requires that the other party be capable of decrypting the data. This is possible because the client and the server will agree on an encryption key during the negotiation phase when the client attempts a secure connection. However, if an attacker (commonly referred to as a Man-In-The-Middle or MitM) masquerades as the server, this encryption key will be negotiated with them and not the target server. This would allow the attacker or MitM to decrypt all the data sent by the client. Obviously, we therefore need the second goal - the ability to verify the identity of the server that the client is communicating with. Without that verification check, we have no way of telling the difference between a genuine target server and an attacker.

So, all four of the above security goals MUST be met before a secure communication can take place. They each work to perfectly complement the other three goals and it is the presence of all four that provides reliable and robust Transport Layer Security.

Aside from the technical aspects of how Transport Layer Security works, the other facet of securely exchanging data lies in how well we apply that security. For example, if we allow a user to submit an application's login form data over HTTP we must then accept that an MitM is completely capable of intercepting that data and recording the user's login data for future use.

This chapter covers the issue of Transport Layer Security from three angles.

* Between a server-side application and a third-party server.
* Between a client and the server-side application.
* Between a client  and the server-side application using custom defenses.

The first addresses the task of ensuring that our web applications securely connect to other parties. Transport Layer Security is commonly used for web service APIs and many other sources of input required by the application.

The second addresses the interactions of a user with our web applications using a browser or some other client application. In this instance, we are the ones exposing a secured URL and we need to ensure that that security is implemented correctly and is not at risk from being bypassed.

The third is a curious oddity. Since SSL/TLS has a reputation for not being correctly implemented by programmers, there have been numerous approaches developed to secure a connection without the use of the established SSL/TLS standards. An example is OAuth's reliance on signed requests which do not require SSL/TLS but offer some of the defences of those standards (notably, encryption of the request data is omitted so it's not perfect but a better option than a misconfigured SSL/TLS library).

Before we get down to those specific categories, let's first take a look at Transport Layer Security in general as there is some important basic knowledge to be aware of before we get down into nuts and bolts with PHP.

Definitions & Basic Vulnerabilities
===================================

Transport Layer Security is a generic title for securing the connection between two parties using encryption, identity verification and so on.


SSL/TLS From PHP (Server to Server)
===================================

As much as I love PHP as a programming language, the briefest survey of popular open source libraries makes it very clear that Transport Layer Security related vulnerabilities are extremely common and, by extension, are tolerated by the PHP community for absolutely no good reason other than it's easier to subject users to security violations than fix the underlying problem. This is backed up by PHP itself suffering from a very poor implementation of SSL/TLS in PHP Streams which are used by everything from socket based HTTP clients to the ``file_get_contents()`` and other filesystem functions. This shortcoming is then exacerbated by the fact that the PHP library makes no effort to discuss the security implications of SSL/TLS failures.

If you take nothing else from this section, my advice is to make sure that all HTTPS requests are performed using the CURL extension for PHP. This extension is configured to be secure by default and is backed up, in terms of expert peer review, by its large user base outside of PHP. Take this one simple step towards greater security and you will not regret it. A more ideal solution would be for PHP's internal developers to wake up and apply the Secure By Default principle to its builtin SSL/TLS support.

My introduction to SSL/TLS in PHP is obviously very harsh. Transport Layer Security vulnerabilities are far more basic than most security issues and we are all familiar with the emphasis it receives in browsers. Our server-side applications are no less important in the chain of securing user data.

Let's examine SSL/TLS in PHP in more detail by looking in turn at PHP Streams and the superior CURL extension.

PHP Streams
-----------

For those who are not familiar with PHP's Streams feature, it was introduced to generalise file, network and other operations which shared common functionality and uses. In order to tell a stream how to handle a specific protocol, there are "wrappers" allowing a Stream to represent a file, a HTTP request, a PHAR archive, a Data URI (RFC 2397) and so on. Opening a stream is simply a matter of calling a supporting function with a relevant URL which indicates the wrapper and target resource to use.

.. code-block:: php

    file_get_contents('file:///tmp/file.ext');

Streams default to using a File Wrapper, so you don't ordinarily need to use a file:// URL and can even use relative paths. This should be obvious since most filesystem functions such as ``file()``, ``include()`` and ``require_once`` all accept stream references. So we can rewrite the above example as:

.. code-block:: php

    file_get_contents('/tmp/file.ext');

Besides files, and of relevance to our current topic of discussion, we can also do the following:

.. code-block:: php

    file_get_contents('http://www.example.com');

Since filesystem functions such as ``file_get_contents()`` support HTTP wrapped streams, they bake into PHP a very simple to access HTTP client if you don't feel the need to expand into using a dedicated HTTP client library like Buzz or Zend Framework's ``\Zend\Http\Client`` classes. In order for this to work, you'll need to enable the ``php.ini`` file's ``allow_url_fopen``configuration option. This option is enabled by default.

However, things get interesting when you try the following:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $result = file_get_contents($url);

The above is a simple unauthenticated request to the Twitter API over HTTPS. It also has a serious flaw. PHP uses an SSL Context (ssl:// transport) for requests made using the HTTPS (https://) and FTPS (ftps://) wrappers. The SSL Context offers a lot of settings for SSL/TLS and their default values are completely insecure. The above example can be rewritten as follows to show how a default SSL Context can be plugged into ``file_get_contents()`` as a parameter:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $contextOptions = array(
        'ssl' => array()
    );
    $sslContext = stream_context_create($contextOptions);
    $result = file_get_contents($url, NULL, $sslContext);

As described earlier in this chapter, failing to securely configure SSL/TLS leaves the application open to a Man-In-The-Middle (MitM) attack. PHP Streams are entirely insecure over SSL/TLS by default. So, let's correct the above example to make it completely secure!

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $contextOptions = array(
        'ssl' => array(
            'verify_peer'   => TRUE,
            'cafile'        => __DIR__ . '/cacert.pem',
            'verify_depth'  => 5,
            'CN_match'      => 'api.twitter.com'
        )
    );
    $sslContext = stream_context_create($contextOptions);
    $result = file_get_contents($url, NULL, $sslContext);

Now we have a secure example! If you contrast this with the earlier example, you'll note that we had to set four options which were, by default, unset or disabled by PHP. Let's examine each in turn to demystify their purpose.

* verify_peer
* cafile
* verify_depth
* CN_match

CURL Extension
--------------

Unlike PHP Streams, the CURL extension is all about performing data transfers including its most commonly known capability for HTTP requests. Also unlike PHP Streams' SSL context, CURL is configured by default to make requests securely over SSL/TLS. You don't need to do anything special unless it was compiled without the location of a Certificate Authority cert bundle (e.g. a cacert.pem or ca-bundle.pem file containing the certs for trusted CAs).

Since it requires no special treatments, you can perform a similar Twitter API call to what we used earlier for SSL/TLS over a PHP Stream with a minimum of fuss and without worrying about missing options that will make it vulnerable to MitM attacks.

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $req = curl_init($url);
    curl_setopt($req, CURLOPT_RETURNTRANSFER, TRUE);
    $result = curl_exec($req);

This is why my recommendation to you is to prefer CURL for HTTPS requests. It's secure by default whereas PHP Streams is most definitely not. If you feel comfortable setting up SSL context options, then feel free to use PHP Streams. Otherwise, just use CURL and avoid the headache. At the end of the day, CURL is safer, requires less code, and is less likely to suffer a human-error related failure in its SSL/TLS security.