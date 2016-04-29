#Revisiting How We Design Operating Systems
*Semi-Organized Ramblings of a Disgruntled CS student*

In a number of years developing user-level applications and tinkering with
userspace on my linux desktop, I've found myself dissatisfied with the
underlying design of operating systems and languages. In the end, all code ends
up causing a bunch of bits to flip in RAM and get copied out to external
devices. If that's the case, why can't I use an ArrayList in my C++ program? Or
have a Java class extend the Maybe monad from haskell? Moreover, how can I have
6 different GUI toolkits on my system with different themes for each? When I'm
writing a game, why do I need to be concerned with efficiently rendering the 3d
models of my environment? These tangential concerns and unnecessary divisions
between equivalent functions drive me insane, because they're caused by poor
underlying design.

##Historical Design Choices for Operating Systems

When the very first punch card computers were created, they were Single User,
Single Process. There was no concept of "user" and they only did one thing at a
time, waiting for the operator to load up the next process to be computed after
the current one finished. This was great, as it made time consuming calculations
much less taxing on the humans, freeing them up for higher-level analysis and
optimization of the calculation. This design is the simplest to implement
and write code for, but has limitations in usability.

Eventually, we moved on to the mainframe design. These mainframes often had more
than one terminal with one user per terminal. These mainframes *could* have been
Multiple User, Single Process, where there are many users and each user only
has one process running at a time. Such an architecture would have each process
running independently of every other process, interleaving time slices such that
each user appears to be on a Single User, Single Process machine.

These mainframes quickly grew to be Multiple User, Multiple Process machines,
where each user can have multiple processes running concurrently and interacting
with each other. This interaction is where everything went wrong.

##The actual problem domain

I think concurrent programming strategies have a "cannot see the
forest for all the trees" issue. By assuming that processes must be able to act
like they're running on a sandboxed mainframe back in the 1950's, we introduce
the myriad issues we see in modern programs and operating systems. What if we
took a step back and looked at the final product and the tools at our disposal
to find a better abstraction?

A user on a modern computer expects many things to be happening all at the same
time. Downloading a large file in their web browser while streaming live audio
and video from a twitch stream in another tab and idling on 5 different chat
clients is not unreasonable. Traditionally, we'd have to make at least one
process for each task and have the OS arbitrate disk and network access, as well
as route user input to the correct application, tied together with the user
interface. This jumbled mess of divisions and overlaps in concern create the
concurrency nightmare.

What the user really wants to be able to do is see and hear the twitch stream,
see and respond to chat feeds, and have that file downloading in the background.
The difference between the two statements is that the first presumes that a
certain 'application' must be used to complete a certain task while the second
just lists out the tasks that must be done. When we just look at what a user may
want to have their computer do, we see that everything boils down to
taking data from a source, transforming it, and then sending it to an output.

## In Depth Example

Web browsing? That's actually a feedback loop requring human input to select the 
next webpage to load. However, we can look at the pipeline of loading the webpage, 
which involves the following basic phases: turn a URL into an https connection and
send the request, read the data off the connection and determine the type of document
being sent, parse the document into a tree (known as the Document Object Model),
set up the javascript callbacks, then start rendering the tree into lower and
lower level graphical primitives, eventually reaching the point where a frame
has been rendered and is sent to the compositor which layers it on top of the
other windows on screen. 

Existing web browsers do all this, and do it very well. Here's the thing. What if I 
want to have my web browser also render Markdown and LaTeX files natively? In existing 
browsers, this is a massive undertaking because all of the rendering code is tightly 
coupled to the network code and hard to append to correctly. Instead of that,
we could design a pipeline that routes documents based on their type to
transformations that turn those documents into parse trees that can be rendered
by one rendering toolkit. To add markdown rendering, all we have to do is write
the markdown parser and plumb it in to the existing pipeline.

This modularity and abstraction come at a cost: Every piece of code on the
computer has to agree on the definitions of the types of data being operated on.
Every single piece of code that might have something to do with a "button" has
to agree on what can and can't be done to a button. This is where we dive into a
bit of language design.

## Language design

Right now, much of the comparisons made between languages that sit in similar
classes is based on their standard libraries. This tight coupling of languages
to functionality is actually quite absurd if we step back and think about it.
Programming is all about manipulating symbols in a logical fashion to produce
a certain structure of symbols from some other structure of symbols. Arguing
over the semantics between `List, ArrayList, Vector`, etc. is actually quite
childish. We have mathematical definitions of how these things work and
algorithms to work on them. I could invent a half-dozen different "languages"
that can be used to express the same calculation in 15 different ways. The
important difference between languages is not the runtime environment, but the
grammatical structure, terseness, and aesthetics of the overall expressions. The
actual symbols being manipulated shouldn't matter.

When we separate the grammar of a language from its vocabulary, we get an
amazing amount of flexibility. It no longer makes sense to quibble over `new
ifstream(filename)` versus `new Scanner(new FileInputStream(filename))`, because
there only needs to be one way to access "files", though there is the
possibility of experimenting with multiple methods and selecting one as the
default and having those programs that rely on some specific derivative require
that derivative to exist.

Can this be done? Yes. Will it change how we write programs? Absolutely! Back to
OS design now that this little detail has been cleared up.

## Practicalities of pipelines

Pipes are one of UNIX's best features. Being able to shuffle data from one
source through a series of transformations is pretty sweet. Why don't we see
them more in modern applications? Well, we never extended the idea of the
pipeline beyond text and the terminal. The plight of screen recording software
illustrates this perfectly. With a pipeline mentality, I should just be able to
copy each frame that's rendered to the screen and place it in a file on my hard
drive to be encoded later. Or if I have a hefty enough machine, encoding it on
the fly. This even works for streaming, as the remote server is expecting a
video stream on the socket, which your machine can pull from the entire screen,
a single window, or some container of multiple windows. Your machine can even do
compositing before sending the frames down the socket, overlaying a facecam,
modifying the volume levels, listening for events on a different socket and
inserting animations and sound into the stream.

Right now, this realm is a complete and utter mess, because it is based on the
idea of inter-process communication. Instead we should modularize the operations
required to produce the desired data stream and then consider "applications" as
data pipelines that translate user input into modified data streams.

The thing that makes the pipe/filter abstraction so useful is that each filter
operates independently of any other filter. If there is a dependency, it needs
to be split up into its independent components. Filters also only operate on
their input data, meaning that a piece of data that has had each of its
dependent filters applied to produce their outputs doesn't need to exist any
more and can be added back to the heap. This removes memory management from the
minds of application and language designers and places it back into the realm of
operating system design, where it should be.

Coming back to the independence of filters, they can be scheduled by a simple
time-slice queuing system, as the order of execution of active filters doesn't
matter. Order of execution is handled by the digraph of data dependencies in
the pipelines, not the code being run. Moving this complex problem from code to
data makes it more palatable to human minds who work better with data structures
than logical flow.
