I learned a lot of things in college, but the most important lessons were not facts or algorithms or theories but rather ways of thinking about and approaching problems. One of the strongest lessons that has stuck with me over the years was a phrase that one of my professors, [Dr. Ford][richard-ford], repeated ad nauseam: “coding by coincidence.”

Pragmatic Programmer

Ford wasn’t the one to coin this phrase, it is in fact a quote from [The Pragmatic Programmer][pragmatic-programmer], but it’s something I will always associate with him. He was an advocate of knowing why things work. Sure you can use some framework or API and spin out an application in ten minutes, but what are you going to do when it breaks? Or worse, what are you going to do when someone discovers a vulnerability in your application and you have no idea what’s going on because you don’t actually know what your code is doing? You’re pretty much screwed.

Some may take this idea and run with it to the extreme, advocating that one should never use third-party frameworks because you didn’t write them. That idea is simply ludicrous, and it is in no way what Ford, Hunt & Thomas, nor I are advocating. Third party frameworks and platforms reduce development time, add important functionality and in general increase security (amongst other benefits). What they don’t do, however, is prevent the developer from doing something stupid. Take search algorithms for example, anyone who calls themselves a software engineer, or a programmer, or computer scientist should know the difference between bubble sort, merge sort and quick sort, etc. Most could probably write these from memory (though it might take a little time), but no one does. Why? Because someone has already done it, and they have probably done it better than you could by leveraging some quirk of the processor architecture or by doing some other [insanely clever thing][fisr]. That being said, the onus is still on the developer to know when to use which algorithm and why it’s appropriate to use one over another. Just as importantly, the code in the framework has been tested and generally has more eyes on it than your application will, meaning that off-by-one error that you would have otherwise accidentally introduced deep in your code will have been caught long ago…

Now Ford was a big proponent of learning by doing. One can only sit in a classroom and be talked at for so long before one’s eyes begin to glaze over and the information starts going in one ear and leaving just as quickly through the other. To this end he always structured his classes in such a way that you were given the tools to figure out how to complete an assignment, but never told how to actually do the assignment. This forced you to poke around at the problem and explore it from every angle while trying to come up with a solution. The benefit of this was that you learned a great deal about the concepts and ideas surrounding and supporting the problem set in the assignment. Fortunately he realized that given the information provided in class you would often need help with the assignments, and he was always available to sit down and provide that help given that you were able to state three ways you tried to solve the problem yourself; if you weren’t able to do this he would turn you right around and send you back out the door. Side note: He also had an [active sitting chair][acive-sitting-chair] which he delighted in slowly bouncing up and down on with ever increasing magnitude while you were trying to explain your problem, which just made getting help that much more hilarious/awkward/interesting…

Even after having this drilled into me I’ve still been guilty of simply grabbing APIs and implementing them in my code without really understanding what is going on behind the scenes. I’m not alone here, we all do it, its unavoidable. There’s just not enough time in the day and its simply not possible for a developer to fully understand their stack. The best we can hope for is to know enough about what we are intending to do, and what should be happening inside the code that we are leveraging, be it calls to the runtime or some other API.

The best example of this that I have experienced occurred a few years ago when I was working on Morse, our shared messaging service in [Legion][legion]. To understand what happened you first need a little bit of background. Back in the first half of 2014 we had a very altruistic doctor who only wanted to help out his good buddy the Nigerian Prince… Now I’m not privy to the exact details, but apparently in this variant the victim was asked to download some attachment to facilitate the transaction. As you may have guessed this attachment was full of malware. Now given the way our security architecture is set up there was no possibility of a data breach from something like this, but fortunately (or unfortunately depending on how you look at it) that was not the goal of this malware. Basically it set up a spam bot on the victim’s machine and mailed itself to everyone in the user’s address book and then tried to copy itself to any open machines on the local network. There’s nothing special about type of malware, viruses like it have been around forever, but that’s not what’s important here, what’s important was the response to it.

What resulted from this attack were two primary changes. The first was the blocking of outbound port 25 (SMTP) for all workstations and all servers which were not explicitly allowed to send email. This didn’t affect any of our team’s production systems as our applications were all already leveraging Morse and the Legion servers were already whitelisted given the amount of email traffic they were handling. The second was enabling the Store & Forward feature on the Exchange servers which had the side effect of delaying the acknowledgement of the receipt of the message until one of three events occurred:

1) The message is delivered to an internal mailbox.
2) The message is delivered to the remote mail server.
3) The message hits the Delayed Acknowledgement max timer (defaults to 30 seconds)

This meant that for any messages being sent to users outside of our domain the SMTP client could potentially block for up to 30 seconds (and often did). We resolved this problem in Morse by threading the send request and simply returning a “success” result to the client application immediately, leaving it up to the application to check back with Morse at some later time to determine if there had been any errors.

The other caveat was Exchange’s maximum concurrent requests limit. Basically each client connected to Exchange is only allowed to make a limited number of concurrent requests to the server (the default is 20). This feature had always been active and prior to activating Store & Forward it wasn’t an issue since the message would be acknowledged immediately. However, with Delayed Acknowledgment coming into play, it was now possible to quickly fill up the allotted slots, and when those slots were filled Exchange would fail any additional requests in excess of that limit. At the time we had two Legion servers meaning that we could theoretically send up to 40 messages simultaneously. This was fine given the amount of traffic we were sending and we didn’t think that we’d reach that limit anytime soon. However, just to be on the safe side we set up a system that would allow messages to sit in our database (all of our messaging is logged) in a queued state until a slot was available on one of the servers at which point it would be sent.

This worked fine for the majority of our application email traffic, however the issue that arose came when sending mass email, for example a planned downtime notification to the 100,000 users of our portal. Historically we sent these via an application on one of our workstations (quick, dirty, simple), however this was no longer possible since workstations were no longer allowed to communicate on port 25. The solution was to proxy all of these messages through Morse and we didn’t have any problems with this set up for a long time. We would simply queue up the 100,000 messages in Morse and let them trickle out as slots became available (mass email has a lower priority then real-time email).

You’re probably wondering by now where I’m going with this… Well, as I said, we had sent several of these mass emails successfully, however there was one mailing where our Public Affairs team wanted to include a PDF attachment in the email. Why not just put the PDF out on a server and include a link? Ours is not to question why… Anyways, Morse already had attachment functionality built into it, however it had never been utilized in production. The problem arose when we noticed that about 5% of the emails with PDFs being sent did have the attachment, but the attachment had a size of 0 bytes.

Here’s a snippet of the send code as originally written, can you spot the problem?

```cs	
public bool SendQueuedMessage(...){
    SendMessage(...);
}
 
public bool Send(...){
    int messageid = CreateRecord(...);
    SendMessage(...);
}
 
protected int CreateRecord(..., System.Net.Mail.AttachmentCollection attachments){
    ...
    byte[] bAttachment;
    foreach (System.Net.Mail.Attachment attachment in attachments) {
        bAttachment = new byte[attachment.ContentStream.Length];
        attachment.ContentStream.Read(bAttachment, 0, bAttachment.Length);
 
        ExecuteStoredProcedure("xspAddAttachment", new List() {
            new StoredProcedureParameter(SqlDbType.Int, "messageid", messageid),
            new StoredProcedureParameter(SqlDbType.VarChar, 255, "name", attachment.Name),
            new StoredProcedureParameter(SqlDbType.VarBinary, -1, "attachment", bAttachment)
        });
    }
}
```

It's always your faultThe issue is that ContentStream.Read() is inherited from the System.IO.Stream() class, and this method maintains a variable named Position which is the current position in the stream. Normally one leverages this variable when handling the stream in chunks using buffers and the like. In this case, the entire stream is being read into an array so it can be passed to SQL for logging, thus moving the Position pointer to the end of the stream. Then, when the service goes to actually send the message, the SMTP client takes the attachment object and does its own read on the stream starting at Position and reading to the end of the stream. As Position is already equal to attachement.Length, 0 bytes are read.

Now why is this bug only affecting 5% of the outgoing messages? Because it’s only affecting messages that are being sent immediately as SendMessage() is being called immediately after CreateRecord() which has moved the Position to the end of the stream. In the case of queued messages, the message record is fetched, and the attachment collection built, but the attachments are never read again before being passed to the SMTP client. Basically it turns out I made a dumb and was implicitly assuming that the SMTP client would check the position of the attachment content stream before reading it, but it doesn’t.

Thankfully the problem was easily solved by adding a single line of code:

```cs	
public bool SendQueuedMessage(...){
    SendMessage(...);
}
 
public bool Send(...){
    int messageid = CreateRecord(...);
    SendMessage(...);
}
 
protected int CreateRecord(..., System.Net.Mail.AttachmentCollection attachments){
    ...
    byte[] bAttachment;
    foreach (System.Net.Mail.Attachment attachment in attachments) {
        bAttachment = new byte[attachment.ContentStream.Length];
        attachment.ContentStream.Read(bAttachment, 0, bAttachment.Length);
        attachment.ContentStream.Position = 0;
 
        ExecuteStoredProcedure("xspAddAttachment", new List() {
            new StoredProcedureParameter(SqlDbType.Int, "messageid", messageid),
            new StoredProcedureParameter(SqlDbType.VarChar, 255, "name", attachment.Name),
            new StoredProcedureParameter(SqlDbType.VarBinary, -1, "attachment", bAttachment)
        });
    }
}
```

Different pieces of .NET code were interacting with each other in weird but predictable ways. It was a pretty silly mistake, and if I worked with streams on a more regular basis I probably would never have made it, in fact I would probably have just reset the position by habit and would never have learned what the SMTP client was doing. Fortunately I knew enough about how each piece should be working and what they were each theoretically doing to be able to resolve the issue in about 10 minutes and we were then able to resend the corrected version of the email to the affected users. The lesson is that leveraging frameworks and APIs is great as long as the developer has some awareness of why the things they are using actually work. More importantly though, it is important to acknowledge that if something isn’t working correctly, more often than not the problem is you. You can spend time blaming the framework or the computer, or you can figure out what underlying assumption you’ve made is incorrect.

Know how your application is working, not just that it is. Maybe not the specifics, but at least the theory. Building an application is easy, building it well is hard. Maintaining it… well that’s always an adventure. This is what Ford was trying teach. By forcing us to think for ourselves, to research solutions rather than follow instructions, to be curious about how things work rather than merely accepting that they were, he prepared us for exactly this type of situation: we might not know why it’s not working, but we know enough to figure it out.

Remember, code deliberately, not by coincidence.

[richard-ford]: http://malware.org/
[pragmatic-programmer]: https://www.goodreads.com/book/show/4099.The_Pragmatic_Programmer
[fisr]: https://en.wikipedia.org/wiki/Fast_inverse_square_root
[active-sitting-chair]: https://www.youtube.com/watch?v=kpbQ3FjLdaQ
[legion]: http://erikjolson.com/legion-web-services-framework/

