---
layout:     post
title:      "Java ByteBuffer to String"
subtitle:   "Il nuovo servizio NoSQL Database offerto da Amazon."
date:       2012-02-09 13:45:00
author:     "Roberto Peruzzo"
tags:       java aws dynamodb amazon bytebuffer
comments:   true
---

<p>Un semplice metodo per convertire un oggetto di tipo ByteBuffer in un oggetto String.</p>

<pre><code>
  public String byteBuffer2String(ByteBuffer pckt) {

    // Convert ByteBuffer to String
    byte[] bytearray = new byte[pckt.remaining()];
    pckt.get(bytearray);

    return new String(bytearray);
  }
</code></pre>
