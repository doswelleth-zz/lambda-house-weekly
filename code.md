##### August 19, 2018

# iOS

CORE DATA

#### David Doswell

Core Data is a programmatic interface to persistently store user data on a device.
The purpose of Core Data is to allow a user to save the content they create to their
device for later use, whether or not they are connected to the Internet.

Core Data is a fascinating architecture with a `NSPersistentContainer` which
"simplifies the creation and management of the Core Data stack by handling the creation of the managed object model (NSManagedObjectModel), persistent store coordinator (`NSPersistentStoreCoordinator`), and the managed object context (`NSManagedObjectContext`)."

Our interface can be written in code but it is preferred that we use Apple's Core Data
"Data Model" file option in Xcode to create our model for our persistent container.

This will allow us to write less code 🎉 and leverage our efforts into making the perfect
**Core Data Stack**. That we will do in code below. Out Core Data Stack will be a [singleton](https://en.wikipedia.org/wiki/Singleton_pattern) which will further allow us to
use it throughout our application as a shared instance on the main context to our
`NSManagedObjectContext`.

```
import Foundation
import CoreData

class CoreDataStack {

    // MARK: - Single instance for app

    static let shared = CoreDataStack()

    lazy var container: NSPersistentContainer = {

        // MARK: Load persistent stores
        let container = NSPersistentContainer(name: "MyProjectName")
        container.loadPersistentStores { (_, error) in
            if let error = error {
                fatalError("Error: \(error)")
            }
        }
        return container
    }()

```

Now, let's create a computed property to calculate and return the container's
[viewContext](https://developer.apple.com/documentation/coredata/nspersistentcontainer/1640622-viewcontext, the interface associated with the main queue:

```
// MARK: - Single context instance for container
   var mainContext: NSManagedObjectContext {
       return container.viewContext
   }

```

`viewContext` is associated with the main queue (or thread) and "is created automatically as part of the initialization of the persistent container."

Cool

```

import Foundation
import CoreData

class CoreDataStack {

    // MARK: - Single instance for app

    static let shared = CoreDataStack()

    lazy var container: NSPersistentContainer = {

        // MARK: Load persistent stores
        let container = NSPersistentContainer(name: "MyProjectName")
        container.loadPersistentStores { (_, error) in
            if let error = error {
                fatalError("Error: \(error)")
            }
        }
        return container
    }()

    // MARK: - Single context instance for container
    var mainContext: NSManagedObjectContext {
        return container.viewContext
    }

}

```

---

##### August 12, 2018

# iOS

LOCAL NOTIFICATIONS

#### David Doswell

Notifications are Apple's interface for alerting users of some event, whether
your app is running in the background, the foreground, or is simply inactive.

The content to the user is relevant and thus requires some handling on the part
of the developer. It is important that content is never lost because of a bad
network connection or inactivity.

There are two types of user interface notifications in UIKit: Local Notifications
and Push Notifications.

Local Notifications are programmatic interfaces called from directly within the app.
Push Notifications are notifications that are _pushed_ to a device from a remote
server.

In either event, it is required of the developer to ask permission before sending
notifications to their users. Best practices also include not inundating users with
notifications or (typically but not always) sending notifications immediately upon
app install.

Here's an interface for getting the authorization status of a new user and for
requesting authorization to send an array of interactions to engage a new user:

```
import Foundation
import UserNotifications

class LocalNotificationHelper {

    func getAuthorizationStatus(completion: @escaping (UNAuthorizationStatus) -> Void) {
        UNUserNotificationCenter.current().getNotificationSettings { (settings) in

            DispatchQueue.main.async {
                completion(settings.authorizationStatus)
            }
        }
    }

    func requestAuthorization(completion: @escaping (Bool) -> Void) {
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { (success, error) in

            if let error = error {
                NSLog("Error requesting authorization status for local notifications: \(error)") }

            DispatchQueue.main.async {
                completion(success)
            }
        }
    }

```    

We use [completion handlers](https://grokswift.com/completion-handler-faqs/) to update the status of our request and, in the case of a `requestAuthorization(_:)`, to test the success
or failure of our request.

# GENERAL

INTRODUCING `wb_alloc`

#### William Bundy

I describe wb_alloc as a collection of allocators, but if you aren't used
to thinking of programming as handling memory the uses of this to C may
not be apparent to you. At the heart of wb_alloc is a slim wrapper around
VirtualAlloc. Virtual memory lets you do things that seem like they should
be impossible based on a simple understanding of memory, but end up making
for nice (and hopefully efficient) features.

To understand how and why these things are useful to C programs, I thought
I'd start with some history. This represents my working understanding but
some details might be incorrect.

### A Survey of Memory History

In the old days, you could imagine memory as a contiguous array of bytes.
That was it. At position 0x1234 there was some value, and you moved on
with your life. If you wanted more memory than you had room for... too
bad. If you wanted more memory than you had numbers for... too bad (i.e., if
you had a 16 bit address bus (pointers could hold the values 0-65535), you
were limited to 64k of memory––see the C64 and NES for systems with this
limitation).

Most systems supported "banking": it was possible to bank in or out memory
at specific locations based on some registers, meaning you could access
more memory than you had numbers for, but not more than you had in the
machine. Or... could you? For cartridge-based systems like the NES and
Gameboy, the cart's data would be accessible in main memory, indexed like
any other pointer. Too, some systems would mirror sections of memory:
multiple regions would map onto the same space. Sometimes this was
configurable, so you could change which area views which memory.

This kind of thing necessitates special hardware for managing the CPU's
understanding of the memory it's connected to. This unit is called
a memory management unit, or usually an MMU. As computers got bigger and
printed silicon got cheaper, we got more memory and numerical space to
address it, but for a long time we were stuck with 16 and 32 bit
computers.

I lump these together because the important difference here is
the size of the address bus. A lot of "16 bit" processors had bigger
than 16 bit address busses (the Super Nintendo and Amiga come to mind).

However, it wasn't until the 32 bit era that we really saw virtual memory
develop. 32 bits allows you to address up to about 4 billion (4 gigabytes
if you like SI prefixes), but programmers like their memory and if you
want to run many programs at once it's very possible to run out.

To address this, MMUs got better. Working with the OS, between the two,
they can map virtual addresses (regular old pointers) to physical
addresses (the actual place where memory is stored). This was generally
more important in the 32-bit era than any other time: the demand for
memory on desktop computers far exceeded the amount of memory addressable.

Virtual memory allowed the OS to write the state of background programs to
disk in the page file/swap space, then reload that into memory when
needed, increasing the effective amount of memory considerably.

And then comes the 64-bit era of computing. 64 bits of addressable space
is enough to store about 18 million terabytes, which is more than enough
for now. While we can't actually fit that much RAM into a machine yet, the
4gb overhead is gone and most CPUs and motherboards can support 32-64 gb
of memory just fine. With all of that room, what benefits do we get from
virtual memory now?

### Writing C in 2018

Well, this is just it: with virtual memory, we can pretend that memory
from (potentially) anywhere is a contiguous array. On Windows, it's as
simple as asking VirtualAlloc to reserve some amount of memory, then
asking again to commit individual segments of memory to the reserved
space. As long as you have enough memory in the machine, it can do that
for you. The actual memory we access might be fragmented over physical
memory and the page file, but it's invisible to our program.

If this sounds kinda scary... yeah, it should be. But, also remember: this
is how _all_ programs get memory now. Every program gets its own virtual
memory space with a method like this. This system is integral to stability
(you can't write over other program's memory by accident), 32-bit support
(32 bit programs just get an empty 4gb range of memory to play with that's
actually mapped somewhere in 64 bit space), and is how OS allocators work
behind the scenes.

So: why not use it to make programming easier and programs simpler? This
is what wb_alloc lets you do.   

### Alternatives

The classic way to make a resizable array in C is to use realloc. You have
a pointer to some memory with objects in it, you call realloc with the new
size, and it'll... allocate some new memory and copy your data there?

This would be a very naive realloc. In a better implementation it could
possibly append adjacent chunks in the heap, and leave everything in
place, but you aren't guaranteed that anything in that array is in the
same place it was originally relative to the origin of the array. This is
fine if you allocate each object on the heap and then allocate an array of
pointers (double indirection is fine, sometimes), but if you want flat
storage for a bunch of objects, doing this invalidates all those pointers.

Another alternative would be to not use an array at all: allocate each
object individually and give each object a next field! A linked list!
This works, but often has some problems in practice. Other than being more
annoying to use, walking linked lists is generally worse than iterating
over arrays--especially in the case where you allocate each link on the
heap; the links can be spread out over the heap, leading to cache misses
(i.e. slowness).

### Using wb_alloc

With wb_alloc, creating a resizable array is easy:

```
#define WB_ALLOC_IMPLEMENTATION
#include "wb_alloc.h"

wb_MemoryInfo meminfo;
wb_MemoryArena* arena;
int intCount;
int* head;

void expand(int newIntCount)
{
	wb_arenaPush(arena, sizeof(int) * (newIntCount - intCount));
	intCount = newIntCount;
}

int main(int argc, char** argv)
{
	meminfo = wb_getMemInfo();
	arena = wb_arenaBootstrap(meminfo, wb_Arena_Normal);
	intCount = 4;
	head = wb_arenaPush(arena, sizeof(int) * intCount);

	expand(1000);
	//...and now we have room for 1000 ints instead of 4
}
```

In this example, expand will just work until you run out of memory.

Internally, `int* head` represents the start of the empty virtual space
and calling `wb_arenaPush` commits more space to the array. To the virtual
memory subsystem in your OS and the MMU, these pages might be coming from
anywhere but to your program they're contiguous.

There are a lot of other ways to use the allocators in wb_alloc. The
memory arena works like a linear allocator or a stack allocator, and can
embed extra information in there, too.

I include a memory pool, which allows for perfect alloc/free mechanics on fixed-
size blobs, and a tagged heap inspired by that one Naughty Dog GDC talk that
effectively acts like a bunch of small arenas (thereby saving on syscalls).

You can find the latest version of the library at
https://github.com/WilliamBundy/wb_alloc

### Caveats

You may have noticed that I play up how scary it is for your memory to be
fragmented behind the scenes, and then talk about how linked lists are bad
because they perform poorly with memory fragmentation. Surely, if relying
on virtual memory leads to some amount of memory fragmentation, these
allocators can suffer from the same issues. But on everything?

As far as I know, the answer is "probably not".

This is towards the edge of my knowledge but virtual memory systems work
on page-sized areas of memory, at least. This means that, at worst, on
a modern computer, your memory might be broken up along 4096 byte pages,
but it's not necessarily that simple. For one, the CPU knows about the
virtual pages. Iterating linearly over arrays is considered good because
it lets the pre-fetcher guess at where you'll want to look at next.

The pre-fetcher could be totally cool with finding things in linearly-accessed
virtual addresses, meaning the performance characteristics don't change
unless you miss the MMU's cache, the translation lookaside buffer (or
TLB). However, if you're working on this memory all those pages should be in
there anyway.

The other thing to think about is how dynamic ram actually works. I don't know
the details, but I'm told that DRAM updates in 64kb areas nowadays, which would
mean that's the "real" page size behind the scenes. It would make sense to me if
OSes actually grabbed memory in variously larger page sizes and sub-allocated it to
programs at smaller sizes. Often, huge pages are 2mb, so, for all I know, Windows
is working with those behind the scenes.

---

wb_alloc works on unix-like systems by following the advice of a blog post
I reference in the source. The mmap interface isn't as clean as VirtualAlloc,
and the actual semantics of mmap are a little different: Windows has a "commit
limit", while mmap allows for something called "overcommit".

On Windows, the method is simple: you "reserve" virtual space, "commit" real
space from that, then write to it. You can reserve up to 8 terabytes of memory,
which is almost free, (though it does take up some memory in the kernel), but
there's a very real limit on the amount of memory that can be committed: the
amount of physical memory in your machine plus the maximum size of your page
file.

On Unix systems, overcommit means that memory is committed when you write to it;
when you need more memory than you have, you crash. This has its ups and downs;
it helps use all your memory, but if you do, eventually things just go bad,
potentially without warning. (that may not be strictly true but I've
never tested and I'm not as confident about the unix backend stuff)

See [this blog post](https://blogs.technet.microsoft.com/markrussinovich/2008/11/17/pushing-the-limits-of-windows-virtual-memory/) for more info on Windows' memory limits:

---

The last claim I make with wb_alloc is that it doesn't include any system
headers, which is correct. It manages this by simply embedding the
relevant sections of header into the source. This is great... unless those
change. I put that in largely as a completeness feature: it's super cool
to just be able to include it and not deal with the hell of giant system
headers (it's especially bad with Windows.h), but I highly recommend that
as you integrate wb_alloc you include the relevant headers yourself, in
case, say, a constant changes behind the scenes.

It's also possible to specify your own backend and all the allocators can
function in a "fixed space" mode, where they simply use a linear block of
memory you hand them. As is, I don't think it's possible to implement
these on top of malloc/realloc directly; memory pools and stack arenas
rely on real pointers, so instability there would break them but there's
definitely room to grow with some of these.

[Back](https://www.lambda.house/about)
