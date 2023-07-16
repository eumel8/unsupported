---
layout: post
tag: de
title: Joomla - SSL Feeds unsupported
subtitle: "Kuerzlich stellte ich die Blogs hier auf SSL um. Es ist jetzt also viel sicherer, dass zu Lesen ;-) Leider liessen sich dann die RSS-Feeds nicht mehr mit Feed Display in Joomla einbetten. Es dauerte eine ganze Weile, um rauszufinden, warum das so. In&hellip;"
date: 2017-07-15
author: eumel8
---

Kuerzlich stellte ich die Blogs hier auf SSL um. Es ist jetzt also viel sicherer, dass zu Lesen ;-)
Leider liessen sich dann die RSS-Feeds nicht mehr mit <strong>Feed Display in Joomla</strong> einbetten. Es dauerte eine ganze Weile, um rauszufinden, warum das so. In "libraries/joomla/http/transport/curl.php" kann man
<!-- codeblock lang=php line=1 --><pre class="codeblock"><code>$options[CURLOPT_SSL_VERIFYPEER] = false;</code></pre><!-- /codeblock --> hinzufuegen, und dann gehts schon wieder.
