---
layout:     post
title:      "AWS DynamoDB"
subtitle:   "Il nuovo servizio NoSQL Database offerto da Amazon."
date:       2012-02-13 12:43:00
author:     "Roberto Peruzzo"
header-img: "img/2012-02-13/dynamoDB.png"
tags:       java aws dynamodb amazon
---

<p>Il 19 gennaio scorso Amazon ha annunciato l'introduzione di un nuovo servizio
  <em>NoSQL Database</em>: sto parlando di <strong>DynamoDB</strong>.
  L'aspetto che più mi interessa tra i vantaggi presentati dall'
  <a href="http://www.allthingsdistributed.com/2012/01/amazon-dynamodb.html" target="_blank">articolo
  scritto nel blog di Werner Vogels</a> è quello della velocità: <strong>riduzione
  della latenza</strong> per garantire uno throughput elevato.</p>

<p>Per ora sto eseguendo alcuni test di scrittura con un'account <em>Amazon Free Tier</em>.
  I risultati non sono entusiasmanti, molto probabilmente dovuti ai limiti
  imposti (max 5 write per second). I miei test procederanno e sarò lieto di
  pubblicare qualcosa nei prossimi giorni; nel frattempo vi posto alcune righe
  di codice per poter interagire con DynamoDB.</p>

<h2 class="section-heading">Creare una tabella</h2>
<pre><code>
  // Get PropertiesCredentials
  InputStream credentialsAsStream = Thread.currentThread().getContextClassLoader()
    .getResourceAsStream("AwsCredentials.properties");
  AWSCredentials credentials = new PropertiesCredentials(credentialsAsStream);

  // Activate the Amazon SimpleDB Client
  this.dynamoDB = new AmazonDynamoDBClient(credentials);

  // Create a table with a primary key named 'name', which holds a string
  CreateTableRequest createTableRequest = new CreateTableRequest()
    .withTableName(tableName)
    .withKeySchema(new KeySchema(new KeySchemaElement()
    .withAttributeName("messageId").withAttributeType("S")))
    .withProvisionedThroughput(new ProvisionedThroughput()
    .withReadCapacityUnits(10L).withWriteCapacityUnits(5L));

  tableDescription = this.dynamoDB.createTable(createTableRequest).getTableDescription();
</code></pre>

<h2 class="section-heading">Salvare un record</h2>
<pre><code>
  // Save item
  Map item = newItem(
    recordId, clientId, body, System.currentTimeMillis());
  PutItemRequest putItemRequest =
    new PutItemRequest(this.table, item);
  PutItemResult putItemResult = dynamoDB.putItem(putItemRequest);

  /**
   * Create item object.
   */
  private Map<String, AttributeValue> newItem(
    String messageId, String clientId, String body, long created) {

    Map<String, AttributeValue> item =
      new HashMap<String, AttributeValue>();

    item.put("messageId", new AttributeValue(messageId));
    item.put("clientId", new AttributeValue(clientId));
    item.put("body", new AttributeValue(body));
    item.put("created", new AttributeValue().withN(Long.toString(created)));

    return item;
  }
</code></pre>
