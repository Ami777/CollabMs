# collab-ms - simple micro-services for Node

Simple, yet complete micro-services library for Node. Zero dependencies, high-quality typed code, simple flow, tree-like structure support, optional Promises, load balancing and queues.

## The idea

I created this library to be able to create complex tree-like-structure micro-services architecture in my Node apps. The idea was to create simple, yet complete solution with good quality typed code (TypeScript & ES6, however you can use this library in pure JS without any additional steps). ZERO external dependencies.

The name in shortcut for "collaboration". In this library we think about micro-services structure as about company. In fact, few concepts are based on management theory. We think about each level as about collaborators: Managers and Workers. Typically, first level is called CEO and is Manager. Each mid-level is both Manager and Worker. Last level is usually Worker.

[@TODO GRAFIKA]

## Features

*   Clean and typed ES6/TypeScript code (pure JS supported without any additional steps)
*   Small codebase, ZERO external dependencies!
*   Manager-Worker, with tree-like-structure support
*   Simple flow - you may communicate one level up or one level down
*   Send non-Promised messages (without any context)
*   Send Promised messages and wait for answer/error
*   Add auto balancing and queues
*   Combine normal messages, Promised messages and auto balancing how you need to
*   Spawn new forks of process, communicate via built-in ICS (well, you don't need to go into details, this is transparent for you)

## Basic concepts

The communication is done in Manager-Worker manner. When you run your process you usually want to spawn main Manager - we call it CEO Manager. This Manager may spawn another Workers and Managers. This is done automatically by forking your main process. It's up to you to decide how to use collab-ms - we do not force you to anything. Here we will cover some basic about communication and our proposition of use.

### Terminology:

*   **Manager** - type of process which may create and manage Workers. It may send messages to Workers and receive answers. It can also receive non-promised messages (without any context).

*   **Worker** - type of process which may receive messages and optionally answer them.

*   **CEO** - this is how we call one main Manager at the top of the structure.

*   **Balancer** - special subtype of Manager. It can create "balanced Workers" and send jobs to them equally. It is also able to keep queue if every Worker is busy.

*   **Normal non-Promised message** - type of message between Manager and Worker. This is message that is sent without any request and without any expectation of answer.

*   **Normal Promised message** - type of message between Manager and Worker. This is message that expects an answer or is answer for another request. May be sent only by Manager to Worker and produces Promises/A+ as the result of sending method.

*   **Work-done non-Promised message** - special case. Mix of normal non-Promised message with Balancer. Worker can inform Balancer it's work is done with it.

*   **Work-done Promised message** - special case. Mix of normal Promised message with Balancer. Worker can inform Balancer it's work is done with it.

## Getting started and tutorials

First, Manager and structure. Then Worker. Then communication. Then some fun.

First, let's create new project with index.js file. Then install collab-ms from npm:

@TODO!

Add library to your code:
```es6
const collab = require('collab');
```

We need to create basic structure. First, we will create first Manager - CEO:
```es6
const ceo = new collab.Manager();
```

This is it. This is all you need to build basic Manager which then can create new Workers and send messages. We cannot receive normal messages yet.

We will use one single file to manage all of the possible processes types. To get type of process spawned we can use collab.getMyRole() and collab.isCEO() functions. Watch out! This is using command line parameters to determinate type. So if you run index.js with any parameters detecting CEO may not work as expected.

Let's use this knowledge to run different job in CEO and in Worker. Modify your const ceo... code:
```es6
if (collab.isCEO()){

 //CEO code

 const ceo = new collab.Manager();

} else {

 //Worker code

}
```

In the else part of code we will create Worker instance:
```es6
if (collab.isCEO()){

 //CEO code

 const ceo = new collab.Manager();

} else {

 //Worker code

 const worker = new collab.Worker();

}
```

Worker can now send messages to closest Manager (it will be CEO in our simple case), but we cannot read any messages from Manager (CEO) yet. Also, Worker is not yet spawned. Let's spawn it using Manager.newWorker() method.

While forking process to create new Worker we have a lot of options. But it's simplest form in just enough for us right now. Let's add this code to CEO code:
```es6
ceo.newWorker('workerExample');
```

This will fork new process with type 'workerExample' (this is our name, it can be anything, but it may contain only letters). 

Let's add console.log() in both parts and now the complete code looks like this:
```es6
const collab = require('./collab.js');

const ceo = new collab.Manager();
if (collab.isCEO()){
 //CEO code
 console.log('CEO is spawned.');
 const ceo = new collab.Manager();
 ceo.newWorker('worker');
} else {
 //Worker code
 console.log('Worker is spawned.');
 const worker = new collab.Worker();
}
```

Run it with:
```console
node index
```

And you should see this:
```console
CEO is spawned.
Worker is spawned.
```

Node won't finish by itself, click Ctrl+C to kill CEO with all of the spawned processes (Worker in our case). Remove console.log() lines.

Now it's time to add communication. We will try something simple - normal non-Promised messages. We need to modify our code to pass callback function for massages in Manager as first argument in constructor. Callback will receive 3 arguments, but we only need 2 right now. Add this callback:
```es6
function (worker, data) {

 console.log('Message', data, 'from', worker.name);

}
```

We also need to add very similar code to Worker constructor. We just want 1 argument which is data:
```es6
function (data) {

 console.log('Message', data, 'from Manager');

}
```

You can then try to send message from Worker to Manager using Worker.send() method in Worker part of code:
```es6
worker.send('Hi boss!');
```

You can send whatever you want - it may be primitive like in our example or more complex object.

Our code is now:
```es6
const collab = require('./collab.js');

const ceo = new collab.Manager();
if (collab.isCEO()){
 //CEO code
 const ceo = new collab.Manager(function (worker, data) {
 console.log('Message', data, 'from', worker.name);
 });
 ceo.newWorker('workerExample');
} else {
 //Worker code
 const worker = new collab.Worker(function (data) {
 console.log('Message', data, 'from Manager');
 });
 worker.send('Hi boss!');
}
```

When we run it we get:

```console
Message Hi boss! from WORKEREXAMPLE #1
```

As you can see, everything is working as expected. Also, you can see name automatically given to Worker by Manager. This name is unique only for forking Manager, not in the whole structure.

Manager can send messages, too. This will take additional steps as Manager may have lots of Workers. Unlike Worker which have only one Manager. We need to access our Worker from Manager using one of these options:

*   Assign to variable value returned by Manager.newWorker() (WorkerInfo)
*   Find Worker by it's name (will also return null or WorkerInfo)
*   Find all Workers by type (will return array of WorkerInfos).

We will use first option right now. So we modify creation line to:
```es6
const workerExample = ceo.newWorker('workerExample');
```

Now we can simply call WorkerInfo.send() method:
```es6
workerExample.send('Hi worker!');
```

The complete code is now:
```es6
const collab = require('./collab.js');

const ceo = new collab.Manager();
if (collab.isCEO()){
 //CEO code
 const ceo = new collab.Manager(function (worker, data) {
 console.log('Message', data, 'from', worker.name);
 });
 const workerExample  = ceo.newWorker('workerExample');
 workerExample.send('Hi worker!');
} else {
 //Worker code
 const worker = new collab.Worker(function (data) {
 console.log('Message', data, 'from Manager');
 });
 worker.send('Hi boss!');
}
```

And it's result is:
```console
Message Hi worker! from Manager
Message Hi boss! from WORKEREXAMPLE #1
```

That's all. You have basic communication done. We will now do some interesting stuff.

### Promises and some real work to do

This will modify previous example. We will use Promised messages to simplify the flow of our code. You may find Bluebird library helpful for Promises. We we also request Worker to do some real work - to add two numbers.

CEO doesn't need callback function as we won't use non-Promised mesages. However, you can still use them and mix both methods. It can be helpful to make your code readable but also to be able to receive non-Promised messages about the state of the Worker etc.

We will also use WorkerInfo.sendWithPromise() method instead of the WorkerInfo.send(). Result of WorkerInfo.sendWithPromise() is Promise. We will use it. CEO code part is now:
```es6
 //CEO code
 const ceo = new collab.Manager();
 const workerExample  = ceo.newWorker('workerExample');
 workerExample.sendWithPromise({a: 2, b: 3}).then(function(result){
 console.log('Result of 2+3 from Worker is', result);
 }).catch(function(err){
 console.log('Error in Worker', err);
 });
 ```

Now, in the Worker code part we also need to modify arguments in constructor. We don't need first argument for now (we will change callback function into null). Second argument is also callback function but for Promised messages. It will be called like this: (data, resolve, reject). Watch out! resolve and reject arguments are not the same are standard Promises/A+ ones (they have second argument, not needed now).

Let's modify Worker code:
```es6
 //Worker code
 const worker = new collab.Worker(null, function (data, resolve, reject) {
 resolve( data.a + data.b );
 });
 ```

Run the code, you should see this:
```console
Result of 2+3 from Worker is 5
```

Promises in collab-ms are easy and good-looking, aren't they?

### Complex structure communication

Now we will try to make more complex structure. This will be CEO Manager-Manager/Worker-Worker. It looks like this, because usually mid-level manager is both Manager and Worker. Then we will try to communicate between closest processes (CEO-Manager and Manager-Worker). At the end we will take a look at passing messages up and down in structure (CEO-Worker and answer Worker-CEO).

To create our structure we need to modify and simplify our code like this:
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()){
 case '': //CEO
 const ceo = new collab.Manager();
 const midLevelManager  = ceo.newWorker('midLevelManager');

        break;
    case 'midLevelManager': //Mid-level Manager
        const midWorker = new collab.Worker();
        const manager = new collab.Manager();
        const newWorker  = ceo.newWorker('worker');

        break;
    case 'worker': //Worker
        const worker = new collab.Worker();

        break;
}
```

Take a look - we use collab.getMyRole() function to get role of the process. Empty string means we want CEO. midLevelManager needs to create both: Worker and Manager instances. This code set ups our structure correctly.

Let's add some simple Promises communication between CEO-Manager and Worker-Manager:
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()){
    case '': //CEO
        const ceo = new collab.Manager();
        const midLevelManager  = ceo.newWorker('midLevelManager');
        midLevelManager.sendWithPromise('Hi.').then(function(data){
            console.log('< Answer', data, 'from mid-level Manager');
        });
        break;
    case 'midLevelManager': //Mid-level Manager
       const midWorker = new collab.Worker(null, function(data, resolve, reject){
           console.log('> Message', data, 'from CEO');
           resolve(data);
       });
       const manager = new collab.Manager();
       const newWorker  = manager.newWorker('worker');
       newWorker.sendWithPromise('Yo!').then(function(data){
           console.log('< Answer', data, 'from Worker');
       });

        break;
    case 'worker': //Worker
        const worker = new collab.Worker(null, function(data, resolve, reject){
            console.log('> Message', data, 'from mid-level Manager');
            resolve(data);
        });

        break;
}
```

Result is:

```console
> Message Hi. from CEO
< Answer Hi. from mid-level Manager
> Message Yo! from mid-level Manager
< Answer Yo! from Worker
```

Now we can try to pass messages from CEO to last Worker and answer from last Worker to CEO. It requires only some simple logic, we will also remove previous messages for clarity (but the logics let us send all of the messages):
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()){
    case '': //CEO
        const ceo = new collab.Manager();
        const midLevelManager  = ceo.newWorker('midLevelManager');
        midLevelManager.sendWithPromise('Tell Worker to come to my office.').then(function(data){
            console.log('< Answer', data, 'from Worker');
        });
        break;
    case 'midLevelManager': //Mid-level Manager
        const manager = new collab.Manager();
        const newWorker  = manager.newWorker('worker');

        const midWorker = new collab.Worker(null, function(data, resolve, reject){
            if (data.indexOf('Worker') > -1){
                newWorker.sendWithPromise('Boss told ya: ' + data).then(resolve).catch(reject);
            } else {
                console.log('> Message', data, 'from CEO');
                resolve(data);
            }
        });

        break;
    case 'worker': //Worker
        const worker = new collab.Worker(null, function(data, resolve, reject){
            if (data.indexOf('Worker') > -1){
                 console.log('> Message', data, 'from Boss passed by mid-level Manager');
                resolve('I\'m on my way CEO.');
            } else {
                console.log('> Message', data, 'from mid-level Manager');
                resolve(data);
            }
        });

        break;
}
```

The result it:
```console
> Message Boss told ya: Tell Worker to come to my office. from Boss passed by mid-level Manager
< Answer I'm on my way CEO. from Worker
```

### Auto balancing

This is the last tutorial example. Here we will use special Balancer subtype of Manager. We will build structure like this: CEO-Balancer-Worker, Worker, Worker.

[@TODO OBRAZEK]

We will tell Balancer to let each of 3 Workers to make 2 jobs at once. Each Worker will finish it's job after 2 seconds. Then, CEO will tell Balancer to give Workers 10 jobs.

First we will use simple communication (work-done non-Promised messages). CEO will tell Balancer to add jobs. Balancer will give jobs to Workers. When Worker is done it will send result to Balancer and it will output it.

Then we will use Promises for all the communication. The flow is almost the same, CEO will also tell Balancer to add jobs but CEO will get an answer from Balancer when all the jobs are done. We will need Bluebird library for this!

First, let's build our structure:
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()){
    case '': //CEO
        const ceo = new collab.Manager();
        const midLevelBalancer  = ceo.newWorker('balancer');

        break;
    case 'balancer': //Balancer Manager
        const manager = new collab.Balancer();
        const midWorker = new collab.Worker();
        for ( let i = 0; i < 3; i++ )
            manager.newBalancedWorker('worker', 2);

        break;
    case 'worker': //Worker
        const worker = new collab.Worker();

        break;
}
```

Take a look. To add Worker in Balancer we use special method Balancer.newBalancedWorker(). Second argument is maximum number of jobs that this Worker can do at the same time. Now, let CEO tell Balancer to add the jobs and let's do it.

CEO and Balancer parts looks now like this:
```es6
    case '': //CEO
        const ceo = new collab.Manager();
        const midLevelBalancer  = ceo.newWorker('balancer');
        midLevelBalancer.send('add10jobs');

        break;
    case 'balancer': //Balancer Manager
        const manager = new collab.Balancer();
        const midWorker = new collab.Worker(function(data){
            if (data == 'add10jobs'){
                for ( let i = 0; i < 10; i++ )
                    manager.addJob();
            }
        });
        for ( let i = 0; i < 3; i++ )
            manager.newBalancedWorker('worker', 2);

        break;
```

Now, we will add some test code to Worker:
```es6
        const worker = new collab.Worker(function(){
            setTimeout(function(){
                worker.sendWorkDone('Done!');
            }, 2000);
        });
```

Can you see the Worker.sendWorkDone() method? It is used to inform Balanced that one job is finished.

And code to receive this message in Balancer, we will modify Balancer's constructor:
```es6
        const manager = new collab.Balancer(function(worker, data){
console.log('Message ',data, 'from', worker.name);
        });
```

Ready code looks like this:
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()){
    case '': //CEO
        const ceo = new collab.Manager();
        const midLevelBalancer  = ceo.newWorker('balancer');
        midLevelBalancer.send('add10jobs');

        break;
    case 'balancer': //Balancer Manager
        const manager = new collab.Balancer(function(worker, data){
            console.log('Message ',data, 'from', worker.name);
        });
        const midWorker = new collab.Worker(function(data){
            if (data == 'add10jobs'){
                for ( let i = 0; i < 10; i++ )
                    manager.addJob();
            }
        });
        for ( let i = 0; i < 3; i++ )
            manager.newBalancedWorker('worker', 2);

        break;
    case 'worker': //Worker
        const worker = new collab.Worker(function(){
            setTimeout(function(){
                worker.sendWorkDone('Done!');
            }, 2000);
        });

        break;
}
```

And the result after about 4 seconds is:
```console
Message  Done! from WORKER #2
Message  Done! from WORKER #1
Message  Done! from WORKER #2
Message  Done! from WORKER #1
Message  Done! from WORKER #3
Message  Done! from WORKER #3
Message  Done! from WORKER #2
Message  Done! from WORKER #1
Message  Done! from WORKER #2
Message  Done! from WORKER #1
```

This is what Balancer is doing after adding or finishing the job:

1.  When new job is added it checks if there is any free (number of current jobs < max. jobs) Worker
2.  If there is no free room, new job is added to queue
3.  When there is any free Worker, Balancer will look for the Worker with smallest number of current jobs
4.  Job is then assigned to this Worker.

Let's change this code to use Promises, it will be a little bit more complex and will use Promise.all():
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()){
    case '': //CEO
        const ceo = new collab.Manager();
        const midLevelBalancer  = ceo.newWorker('balancer');
        midLevelBalancer.sendWithPromise('add10jobs').then(function(data){
            console.log('Balancer told CEO that all jobs are done!', data);
        });

        break;
    case 'balancer': //Balancer Manager
        const manager = new collab.Balancer();
        const midWorker = new collab.Worker(null, function(data, resolve, reject){
            if (data == 'add10jobs'){
                let promises = [];
                for ( let i = 0; i < 10; i++ )
                    promises.push(manager.addJobWithPromise());
                Promise.all(promises).then(resolve).catch(reject);
            }
        });
        for ( let i = 0; i < 3; i++ )
            manager.newBalancedWorker('worker', 2);

        break;
    case 'worker': //Worker
        const worker = new collab.Worker(null, function(data, resolve, reject){
            setTimeout(function(){
                resolve('Done!', true);
            }, 2000);
        });

        break;
}
```

There is one new thing here. Look, when Worker is done it is passing true as a second argument to resolve function. This is very important in our code, because it will tell Manager it should treat answer as work-done Promised message.

The result after about 4 seconds is:
```console
Balancer told CEO that all jobs are done! [ 'Done!',  'Done!',  'Done!',  'Done!',  'Done!',  'Done!',  'Done!',  'Done!', 'Done!',  'Done!' ]
```

Now we have all: Promises, balancing, queue, tree structure. In just a few lines of code.

## Reference / documentation
This is typing file for collab-ms, it is also used as a reference/documentation. If you use collab-ms with TypeScript you have this docs in your IDE, too.
```typescript
import { ChildProcess } from "child_process";
declare module Collab {
    interface NormalSendFunction {
        /**
         * @param data Any data you want to pass.
         */
        (data?: any): void;
    }
    interface ResolveFunction {
        /**
         * @param data Any data you want to pass.
         */
        (data?: any): void;
    }
    interface ResolveBalancedFunction {
        /**
         * @param data Any data you want to pass.
         * @param sendWorkDone False (default) means this will be sent as normal Promise answer. Set to True to send as work-done Promise answer.
         */
        (data?: any, sendWorkDone?: boolean): void;
    }
    interface RejectFunction {
        /**
         * @param error Error information to send.
         */
        (error?: any): void;
    }
    interface RejectBalancedFunction {
        /**
         * @param error Error information to send.
         * @param sendWorkDone False (default) means this will be sent as normal Promise answer. Set to True to send as work-done Promise answer.
         */
        (error?: any, sendWorkDone?: boolean): void;
    }
    interface WorkerMsgClbFunction {
        /**
         * @param worker Current Worker info object.
         * @param data Data passed from Worker.
         * @param send This is shortcut to worker.send() function for quick answers.
         */
        (worker?: WorkerInfo, data?: any, send?: NormalSendFunction): void;
    }
    interface ManagerMsgClbFunction {
        /**
         * @param data Data passed from Manager.
         * @param send This is shortcut to Worker.send() function for quick answers.
         */
        (data?: any, send?: NormalSendFunction, sendWorkDone?: NormalSendFunction): void;
    }
    interface ManagerPromisedMsgClbFunction {
        /**
         * @param data Data passed from Manager.
         */
        (data?: any, resolve?: ResolveBalancedFunction, reject?: RejectBalancedFunction): void;
    }
    /**
     * Information about Worker.
     */
    interface WorkerInfo {
        /**
         * Name given automatically.
         */
        name?: string;
        /**
         * Type name given by you.
         */
        type?: string;
        /**
         * Options passed by you when forking.
         */
        options?: any;
        /**
         * Internal ChildProcess.
         */
        process?: ChildProcess;
        /**
         * Your internal data.
         */
        data?: any;
        /**
         * Function to send non-Promised message.
         */
        send?: NormalSendFunction;
        /**
         * Function to send Promised message.
         */
        sendWithPromise?: NormalSendFunction;
    }
    /**
     * This is internal structure used for Promises.
     */
    interface Promises {
        id: number;
        resolve?: ResolveFunction;
        reject?: RejectFunction;
    }
    class Manager {
        protected onWorkerMessage: WorkerMsgClbFunction;
        protected promiseIdx: number;
        protected promises: Promises[];
        protected workers: WorkerInfo[];
        /**
         * Class constructor for Manager - CEO and mid-level managers.
         * @param onWorkerMessage Callback which will run when non-Promised message arrives to Manager from Worker.
         */
        constructor(onWorkerMessage?: WorkerMsgClbFunction);
        protected onMessage(worker: WorkerInfo, data: any): void;
        /**
         * Adds new Worker using child_process.fork() and links it with this Manager. This will return WorkerInfo instance with the possibilities to send messages and with unique name field.
         * @param type String with name of type of Worker (for example 'worker' or 'readNode'). MUST BE ONE WORD, ONLY LETTERS.
         * @param moduleOrFile Module or file to run (to be used as first parameter in child_process.fork()).
         * @param options Options to pass to the Worker - may be anything.
         * @param data Data about this Worker to store in this Manager. May by anything.
         * @param forkOpts Any fork options (options : ForkOptions) you may use with child_process.fork().
         */
        newWorker(type: string, moduleOrFile?: string, options?: any, data?: any, forkOpts?: any): WorkerInfo;
        /**
         * Find WorkerInfo by Worker name.
         * @param name Name of Worker.
         */
        getWorker(name: string): WorkerInfo;
        /**
         * Find array of WorkerInfo by Worker type.
         * @param type Type of Worker.
         */
        getWorkers(type: string): WorkerInfo[];
        private _buildFuncSendWithPromise(process);
    }
    class Balancer extends Manager {
        private queue;
        /**
         * Class constructor for Balancer Manager - mostly it will be special mid-level manager.
         * @param onWorkerMessage Callback which will run when non-Promised message arrives to Manager from Worker.
         */
        constructor(onWorkerMessage?: WorkerMsgClbFunction);
        private onQueueCheckInterval();
        private findMostFreeWorker();
        /**
         * Adds new Worker using child_process.fork() and links it with this Manager. This is special type of Worker which will be managed and balanced by this Balancer. For more information refer to Manager.newWorker() docs.
         * @param type String with name of type of Worker (for example 'worker' or 'readNode'). MUST BE ONE WORD, ONLY LETTERS.
         * @param moduleOrFile Module or file to run (to be used as first parameter in child_process.fork()).
         * @param maxJobsAtOnce Maximum number of jobs that this Worker should do at once.
         * @param options Options to pass to the Worker - may be anything.
         * @param data Data about this Worker to store in this Manager. May by anything.
         * @param forkOpts Any fork options (options : ForkOptions) you may use with child_process.fork().
         */
        newBalancedWorker(type: string, maxJobsAtOnce: number, moduleOrFile?: string, options?: any, data?: any, forkOpts?: any): WorkerInfo;
        /**
         * Adds job to do by some of the best-suited Worker. Best-suited Worker is the one with the smallest amount of current jobs and with free space for next one. If no Worker can be found the job is queued and when any of the Workers will be free this job will be executed.
         * @param data Any data you want to pass to the Worker.
         */
        addJob(data?: any): void;
        /**
         * Same as Balancer.addJob() but with Promises.
         * @param data Any data you want to pass to the Worker.
         */
        addJobWithPromise(data?: any): Promise<any>;
        protected onMessage(worker: WorkerInfo, data: any): void;
    }
    class Worker {
        onManagerMessage: ManagerMsgClbFunction;
        onManagerMessageWithPromise: ManagerPromisedMsgClbFunction;
        private type;
        private name;
        private options;
        /**
         * Class constructor for Worker - it will be any worker including mid-level manager.
         * @param onManagerMessage Callback which will run when non-Promised message arrives to Worker from Manager.
         * @param onManagerMessageWithPromise Callback which will run when Promised message arrives to Worker from Manager.
         */
        constructor(onManagerMessage?: ManagerMsgClbFunction, onManagerMessageWithPromise?: ManagerPromisedMsgClbFunction);
        /**
         * Reads type name of Worker passed by Manager to this Worker while forking it.
         */
        getType(): string;
        /**
         * Reads options passed by Manager to this Worker while forking it.
         */
        getOptions(): any;
        /**
         * Reads name of Worker passed by Manager to this Worker while forking it.
         */
        getName(): string;
        private onMessage(data);
        private _makeResolveFunc(promiseId);
        private _makeRejectFunc(promiseId);
        /**
         * Sends normal, non-Promised message to closest Manager.
         * @param data Any data you want to pass to the Manager.
         */
        send(data?: any): void;
        /**
         * Sends work-done, non-Promised message to closest Manager. This is usually answer for Balancer Manager.
         * @param data Any data you want to pass to the Manager.
         */
        sendWorkDone(data?: any): void;
    }
    /**
     * Reads type name of Worker passed by Manager to this Worker while forking it or empty string for main CEO process.
     */
    function getMyRole(): string;
    /**
     * Returns true if this is main process.
     */
    function isCEO(): boolean;
}
export = Collab;
```

## Examples
Here you may find some pure examples of usage of collab-ms. Examples are ready-to-use, just copy&paste code into index.js file and run it.

### Basic usage example
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        (new collab.Manager((worker, msg) => {
            console.log('Msg', msg, 'from worker');
        })).newWorker('worker');
        break;

    case 'worker': //Worker
        (new collab.Worker()).send('Hi, ready to work!');
        break;
}
```

### Few Workers at the same level
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        const ceo = new collab.Manager((worker, msg) => {
            console.log('Msg', msg, 'from worker', worker.name);
        });
        for (let i = 0; i < 3; i++) {
            const name = ceo.newWorker('worker').name;

            console.log('New worker is named', name);
        }
        break;

    case 'worker': //Worker
        const worker = new collab.Worker();
        worker.send('Hi, ready to work!');
        break;
}
```

### Basic usage with Promises
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        (new collab.Manager()).newWorker('worker').sendWithPromise('Hi!').then(ans => {
            console.log('Answer from worker:', ans);
        })
        break;

    case 'worker': //Worker
        new collab.Worker(null, (data, resolve, reject) => {
            console.log('Request from CEO:', data);
            resolve('Oh hey!');
        });
        break;
}
```

### Basic Promises with error handling (reject)
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        const ceo = new collab.Manager();

        const mathWorker = ceo.newWorker('mathWorker');
        mathWorker.sendWithPromise({
            a: 2,
            b: 3
        }).then(ans => {
            console.log('Answer from worker', ans);
        }).catch(err => {
            console.log('Error in worker', err);
        });
        break;

    case 'mathWorker': //Worker
        const worker = new collab.Worker(null, (data, resolve, reject) => {
            resolve( data.a + data.b );
        });
        break;
}
```

```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        (new collab.Manager()).newWorker('worker').sendWithPromise('Can you work?').then(ans => {
            console.log('Answer from worker:', ans);
        }).catch(err => {
            console.log('Error in worker:', err);
        });
        break;

    case 'worker': //Worker
        new collab.Worker(null, (data, resolve, reject) => {
            console.log('Request from CEO:', data);
            if (data.indexOf('work') > -1)
                reject('704 Motivation Not Found');
        });
        break;
}
```

### Mix of "async" random input and expected result of Promise call
Both non-Promised and Promissed messages.
```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        //We can receive "random input"
        const ceo = new collab.Manager((worker, data) => {
            console.log('Random input: ping from ' + worker.name);
        });

        const mathWorker = ceo.newWorker('mathWorker');

        //And still we can send Promised call - when we expect answer
        mathWorker.sendWithPromise({
            a: 2,
            b: 3
        }).then(ans => {
            console.log('Answer from worker', ans);
        }).catch(err => {
            console.log('Error from worker', err);
        });
        break;

    case 'mathWorker': //Worker
        const worker = new collab.Worker(null, (data, resolve, reject) => {
            resolve( data.a + data.b );
        });

        setInterval(() => {
            worker.send({}); // Just ping
        }, 1000);
        break;
}
```

### Complex example - load balancing, Promises, tree structure
The idea here is to sum all of the numbers in array multipied by 2. We also create tree structure:
[@TODO IMAGE]

```es6
const collab = require('./collab.js');

switch(collab.getMyRole()) {
    case '': //Main/CEO
        (() => {
            const ceo = new collab.Manager();
            const balancer = ceo.newWorker('balancer');
            balancer.sendWithPromise({
                data: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
            }).then(ans => {
                console.log('Answer from balancer', ans);
            }).catch(err => {
                console.log('Error from balancer', err);
            });
        })();
        break;

    case 'balancer': //Banalcer/queue
        (() => {
            const balancer = new collab.Balancer();
            for (let i = 0; i < 3; i++) {
                balancer.newBalancedWorker('mathWorker', 2);
            }

            //Im also Worker
            const worker = new collab.Worker(null, (data, resolve, reject) => {
                let promises = [];
                data.data.forEach(number => {
                    promises.push(balancer.addJobWithPromise({
                        number
                    }));
                });

                Promise.all(promises).then((numbers) => {
                    const sum = numbers.reduce( ( acc, cur ) => acc + cur.result, 0 );
                    resolve(sum);
                }).catch(reject);
            });
        })();
        break;

    case 'mathWorker': //Worker
        const worker = new collab.Worker(null, (data, resolve, reject) => {
            setTimeout(() => {
                resolve({
                    result : data.number * 2
                }, true);
            }, 2000);
        });
        break;
}
 ```
 
 ### Big tree structure - a lot of levels
 This is usually not good idea, but it works.
 ```es6
 const collab = require('./collab.js');

const structureDepth = 20;
switch(collab.getMyRole()) {
    case '': //Main/CEO
        (() => {
            const ceo = new collab.Manager((worker, data) => {
                console.log('Input message from', worker.name, 'data is', data);
            });

            ceo.newWorker('nextLvlWorker', 'index', 1);
        })();
        break;

    case 'nextLvlWorker':
        (() => {
            //Im also worker
            const meWorker = new collab.Worker();
            const meManager = new collab.Manager((worker, data) => {
                console.log('Passing message from level', meWorker.getOptions()+1, 'to level', meWorker.getOptions());
                data.levels.push(meWorker.getOptions());
                meWorker.send(data);
            });

            console.log('Created nextLvlWorker level', meWorker.getOptions());

            if (meWorker.getOptions() == structureDepth){
                meWorker.send({
                    levels:[structureDepth],
                    info:'OK!',
                });
            } else {
                meManager.newWorker('nextLvlWorker', 'index', meWorker.getOptions()+1);
            }
        })();
        break;
}
```

## TODO
There are a lot of things to do or to add maybe later. The most important is to write **tests** right now to make it really high-qualiy. 
 
## License
MIT licensed.
Created by Jakub Król, IT.focus.
