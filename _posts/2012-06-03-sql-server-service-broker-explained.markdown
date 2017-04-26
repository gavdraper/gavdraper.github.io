---
layout: post
title: SQL Server Service Broker Explained
date: '2012-06-03 12:08:09'
---

<h3>What is Service Broker?</h3> <p>Service Broker is a native SQL Server implementation of message queues.</p> <h3>What are message queues?</h3> <p>Message queues are a way of sending asynchronous messages across boundaries. Different message queue implementations offer varying features but the key features offered by pretty much all of them are…</p> <ul> <li>Guaranteed message delivery, if the target is offline the message will be held and delivered when it comes back online..  <li>Asynchronous communication.  <li>Public API’s.  <li>Transactions.  <li>Backup options.</li></ul> <h3>How do they work?</h3> <p>As you can probably guess from the name when you send a message you typically specify a queue to send it to, this could be either local or remote. If the queue is offline the message will stay in some sort of pending transactions store where it will be sent on when the queue comes back online. At this point there will be some sort of receiver service that processes messages *in order* from the queue and possibly send one back.</p> <p>*Not all implementations guarantee in order processing of messages, MSMQ for example could have messages stored out of order due to one traveling faster over the wire than another. This however is solved in Service Broker with use of conversation groups and message sequence numbers. Also MSMQ could process messages out of order if you are processing messages in multiple threads, again Service Broker solves this by locking conversation groups guaranteeing all messages sent in that group will be processed in order.</p> <h3>Why use them?</h3> <ul> <li>Allow you to decouple dependencies on other applications by communicating through messages.  <li>Allow you to massively scale out your architecture by moving queues/message processors to separate servers as and when needed.  <li>Ability to take offline/upgrade individual parts of the system with minimal impact to the end users.  <li>You can control how and when the messages are processed. For example you may send payment processing messages to a payment queue to be processed overnight when your server is at a lower load.  <li>You can have message processors for the same queue on multiple servers/processes/threads giving you maximum flexibility as to how your queue is processed.</li></ul> <h3></h3> <h3>Why Service Broker?</h3> <p>It’s part of SQL Server so offers a lot of benefits other implementations struggle with. Some of these being….</p> <ul> <ul> <li>Transactions  <li>Backups, as the queues are part of the database all you need to do is backup the database.  <li>Locking of conversation groups&nbsp; to ensure in order processing.  <li>Mirroring.</li></ul></ul> <h3>Time for an example….</h3> <p>I’m going to walk you through implementing a solution to a typical scenario that benefits from the use of Service Broker, the complete project for this example can be downloaded from github at <a href="https://github.com/gavdraper/ServiceBrokerTicketMaster">https://github.com/gavdraper/ServiceBrokerTicketMaster</a>. I recommend you clone the example and have a look at the source code as I’m not going to cover everything here. </p> <p>The scenario for this example is we are building a ticket booking website that will have 1000’s of concurrent users and we need the flexibility to scale quickly.</p> <p>There are 2 components of this system that don’t need to happen in real time and could possible be deferred to periods of low activity, these are the payment processing and ticket printing. </p> <p>This is how it is going to work….</p> <p><a href="/content/images/WPImport/2012/06/image.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="image" border="0" alt="image" src="/content/images/WPImport/2012/06/image_thumb.png" width="518" height="551"></a></p> <ol> <li>A user hits the “Create Booking” button on the web application to create a booking.  <li>The webserver inserts the booking record into the SQL Server database.  <li>The Webserver sends a message to the payment processor target queue to process the payment for that booking.  <li>The payment processor service will pick the message up and process the payment  <ol> <li>If the payment is a success it sends a payment success message back to the payment processor initiator queue.  <li>If it fails it sends a fail message back</li></ol> <li>SQL Server is monitoring the payment initiator queue for payment responses.  <ol> <li>If a payment fails it sets the booking record to payment failed.  <li>If the payment was a success it sets the booking table to payment success and sends a message to the ticket printer target queue.</li></ol> <li>The ticket printing service picks up the message in the ticket printer target queue and prints the ticket.  <ol> <li>If this succeeds it sends a success message back to the ticket printer initiator queue  <li>If it fails it sends a failed message.</li></ol> <li>SQL Server picks up the message from the ticket printer initiator queue and with sets the ticket to printed or failed.</li></ol> <p>With this example its very easy to add payment processing/printing servers as needed. We can also schedule them to start and stop as and when needed meaning we could defer either of them to process overnight if we were becoming limited on resources. This option would have been a lot harder to implement if you were synchronously doing these operations on your webserver/sqlserver.</p> <p>The first thing we need in this example are to define our message types, in this case we have 4 that we need to create Print Request/Response, and Payment Request/Response. You can create them by running the following TSQL…</p><pre class="brush: sql; gutter: false; toolbar: false;">CREATE MESSAGE TYPE [PrintRequest] VALIDATION = WELL_FORMED_XML
CREATE MESSAGE TYPE [PrintResponse] VALIDATION = WELL_FORMED_XML
CREATE MESSAGE TYPE [ProcessPaymentRequest]&nbsp; VALIDATION = WELL_FORMED_XML
CREATE MESSAGE TYPE [ProcessPaymentResponse] VALIDATION = WELL_FORMED_XML</pre>
<p>The only restrictions we are placing on these messages are that they must be well formed XML which is fine because our app is the only app sending messages. If we were communicating with third party apps we can set the messages to validate against an XML Schema to make sure the format is 100% correct.</p>
<p>Next we need to define the queues the messages are going to be sent to/from, in this case we have a queue per message type so that’s 4 queues.</p><pre class="brush: sql; gutter: false; toolbar: false;">CREATE QUEUE PrintInitiatorQueue WITH ACTIVATION
(
	STATUS = ON,
	PROCEDURE_NAME = [BrokerPrintMessageProcessed],
	MAX_QUEUE_READERS = 4,
	EXECUTE AS SELF
)
CREATE QUEUE PrintTargetQueue WITH STATUS = ON
CREATE QUEUE ProcessPaymentInitiatorQueue WITH ACTIVATION
(
	STATUS = ON,
	PROCEDURE_NAME = [BrokerPaymentMessageProcessed],
	MAX_QUEUE_READERS = 4,
	EXECUTE AS SELF
)
CREATE QUEUE ProcessPaymentTargetQueue WITH STATUS = ON</pre>
<p>As you can see in the diagram/workflow&nbsp; at the start of the example the target queues are monitored by separate services but the initiator queues are monitored and processed by SQL Server. You can see in the queue code above for the initiator queues I specify with activation and pass in some extra parameters. The activation is a way of telling service broker to run a procedure/CLR method&nbsp; when a new message appears on the queue. It will only do this&nbsp; if the processing procedure isn't already running or if it decides a second (third/fourth….) procedure would process the queue quicker.</p>
<p>Just to clear things up target queues are where a message is first sent to and initiator queues are where the response is sent, from this point the conversation can go back and forth for as long as is needed until both parties end the conversation.</p>
<p>Ok so we have our queues and message types now we need to create some contracts to define the messages allowed in a conversation…</p><pre class="brush: sql; gutter: false; toolbar: false;">CREATE CONTRACT [ProcessPaymentContract]
(
	[ProcessPaymentResponse] SENT BY TARGET,
	[ProcessPaymentRequest] SENT BY INITIATOR
)
CREATE CONTRACT [PrintContract]
(
	[PrintRequest] SENT BY INITIATOR,
	[PrintResponse] SENT BY TARGET
)
</pre>The last step is to create the actual services…<pre class="brush: sql; gutter: false; toolbar: false;">CREATE SERVICE PrintInitiatorService ON QUEUE PrintInitiatorQueue(PrintContract) 
CREATE SERVICE PrintTargetService ON QUEUE PrintTargetQueue(PrintContract) 
CREATE SERVICE ProcessPaymentInitiatorService ON QUEUE ProcessPaymentInitiatorQueue(ProcessPaymentContract)
CREATE SERVICE ProcessPaymentTargetService ON QUEUE ProcessPaymentTargetQueue(ProcessPaymentContract)
</pre>
<p>We now have everything we need to begin sending messages back and forth. </p>
<p>Lets imagine when a user creates a booking it calls the CreateBooking stored procedure, this procedure creates a booking record in the booking table then sends a message to the ProcesssPaymentTarget queue for a payment app to pick up and process the payment. Sending a message to the ProcessPaymentTarget queue can be done in TSQL like this…</p><pre class="brush: sql; gutter: false; toolbar: false;">CREATE PROCEDURE [dbo].[CreateBooking]
(
 @EventId INT,
 @CreditCard VARCHAR(20)
)
AS

BEGIN TRANSACTION
BEGIN TRY
	INSERT INTO Bookings(EventId,CreditCard)
	VALUES(@EventId,@CreditCard)

	DECLARE @BookingId INT = SCOPE_IDENTITY()

	--Send messagge for payment process
	DECLARE @ConversationHandle UNIQUEIDENTIFIER
		BEGIN DIALOG CONVERSATION @ConversationHandle
		   FROM SERVICE [ProcessPaymentInitiatorService]
		   TO SERVICE 'ProcessPaymentTargetService'
		   ON CONTRACT [ProcessPaymentContract]
		   WITH ENCRYPTION = OFF
		DECLARE @Msg NVARCHAR(MAX) 
		SET @Msg = '<payment><bookingid>' + CAST(@BookingId AS NVARCHAR(10)) + '</bookingid><creditcard>' + @CreditCard + '</creditcard></payment>';
		SEND ON CONVERSATION @ConversationHandle MESSAGE TYPE [ProcessPaymentRequest](@Msg)
	COMMIT
END TRY
BEGIN CATCH
	RAISERROR('Booking Failed',1,1)
	ROLLBACK
	RETURN
END CATCH</pre>
<p>As you can see it’s pretty much a case of specifying the source and destination services with the contract you want to use and you can then send your message.</p>
<p>If you look at the full <a href="https://github.com/gavdraper/ServiceBrokerTicketMaster">example project</a> on github you can see I then have a console app that picks up this message, there are a few ways you can achieve this.</p>
<ul>
<li>You can poll the service broker receive method with a timeout. This means you call receive and specify a timeout duration so it will sit in the receive method until it either times out or a message is received. You can then loop round and keep doing this polling/waiting technique. 
<li>You can subscribe to service broker events on SQL server that will fire when a message is received.</li></ul>
<p>I’m not going to go through all the source of the payment console app as its on github and quite straight forward instead I’m going to show you the BrokerPrintMessageProcessed stored procedure that…</p>
<ol>
<li>Pulls a message off the ProcessPaymentInitiator queue. </li>
<li>Parses the message and updates the booking to say if the payment succeeded.</li>
<li>Sends the end conversation message back to end the payment conversation.</li>
<li>If the payment was a success it will send a message to the PrintTarget queue instructing the booking to have it’s ticket printed.</li></ol>
<p>One thing to note is that when a conversation is over both parties need to send an end conversation message or the conversation will stay alive forever!</p><pre class="brush: sql; gutter: false; toolbar: false;">CREATE PROCEDURE [dbo].[BrokerPaymentMessageProcessed]
AS
DECLARE @ConversationHandle UNIQUEIDENTIFIER
DECLARE @MessageType NVARCHAR(256)
DECLARE @MessageBody XML
DECLARE @ResponseMessage XML

WHILE(1=1)
BEGIN
	BEGIN TRY
		BEGIN TRANSACTION
		WAITFOR(RECEIVE TOP(1)
			@ConversationHandle = conversation_handle,
			@MessageType = message_type_name,
			@MessageBody = CAST(message_body AS XML)
		FROM
			ProcessPaymentInitiatorQueue
			), TIMEOUT 1000
		IF(@@ROWCOUNT=0)
			BEGIN
				ROLLBACK TRANSACTION 
				RETURN
			END
		SELECT @MessageType
		IF @MessageType = 'ProcessPaymentResponse'
			BEGIN
			--Parse the Message and update tables based on contents
			DECLARE @PaymentStatus INT = @MessageBody.value('/Payment[1]/PaymentStatus[1]','INT')
			DECLARE @BookingId INT = @MessageBody.value('/Payment[1]/BookingId[1]','INT')
			UPDATE Bookings SET Bookings.PaymentStatus = @PaymentStatus WHERE Id = @BookingId
			--Close the conversation on the Payment Service
			END CONVERSATION @ConversationHandle				
			--Start a new conversation on the print service
			BEGIN DIALOG CONVERSATION @ConversationHandle
				FROM SERVICE [PrintInitiatorService]
				TO SERVICE 'PrintTargetService'
				ON CONTRACT [PrintContract]
				WITH ENCRYPTION = OFF
			DECLARE @Msg NVARCHAR(MAX) = '<bookingid>' +  CAST(@BookingId AS NVARCHAR(10)) +  '</bookingid>';
			SELECT @ConversationHandle;
			SEND ON CONVERSATION @ConversationHandle MESSAGE TYPE [PrintRequest](@Msg)						
			END
		COMMIT
	END TRY
	BEGIN CATCH
		SELECT ERROR_MESSAGE()
		ROLLBACK TRANSACTION
	END CATCH
END
</pre>
<p>Things to note….</p>
<p>If you look back at the TSQL we used to create the queues you’ll see the above procedure is the one that automatically gets called by the use of activation when a message is received on the PrintInitiatorQueue queue.</p>
<p>WHILE (1=1) : This procedure deliberately runs in an endless loop until a call to receive returns no messages. This is so the whole queued gets processed without having to keep calling this procedure from activation.</p>
<p>The call to receive takes a timeout in this case 1 second, meaning it will wait at this point until a message appears on the queue or the timeout expires. If the timeout expires the @@Rowcount will be zero and we can return from the procedure as there is nothing else in the queue.</p>
<p>The rest should be fairly straight forward as we are just parsing the received message to update our bookings table and then sending the end conversation message back to close the conversation. Once this is done we then open up a new conversation to the PrintTargetQueue for the print service to pick up and handle printing.</p>
<p>The best way to see all this in action is to clone and build the source from <a href="https://github.com/gavdraper/ServiceBrokerTicketMaster">github</a>, then run both the print and payment console apps, once they are running run the webapp and make a booking. You will then be able to see the messages flow from the webapp to the payment console and then to the print console.</p>