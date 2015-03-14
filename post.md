There's been a lot of buzz lately about microservices, and a lot of developers
who I have worked with initially have trouble understanding how to make the 
communication between services work well without a lot of effort in thinking 
about and re-implementing the communication patterns in each service.  The 
idea of microservices strongly discourages code sharing between services, and 
eliminating hard dependencies between them that could easily break when changes 
are made elsewhere in the system.

I've published a Node.js module for doing microservices that communicate over 
RabbitMQ.  Many products at a company I used to work for are built off of the 
module (or the .NET equivalent of it that I also created), and it has proven 
to be stable and perform well.

Here's a quick example of how my module can be used to create a microservice.

Make sure RabbitMQ is installed on the local machine.  It should be using 
the standard configuration, and allow the default guest user.

Now, we can install the Node.js module:

<!-- code lang=bash -->

    npm install microservice-crutch

<!-- /code -->

Lets create a ping-listener service that replies to ping messages that it 
receives from a topic exchange:

<!-- code lang=javascript -->

    var crutch = require('../crutch.js');
    var defaults = {
        defaultExchange: 'topic://example',
        defaultQueue: 'ping-listener',
        defaultReturnBody: false,
        'log.level': 'ERROR',
        pongValue: 'PONG',
    };
    
    crutch(defaults, function(logging, microservices, options, Promise) {
        var log = logging.getLogger('ping-listener');
        log.setLevel('INFO');
        
        return microservices.bind('example.ping', function(mc) {
            return Promise
                .try(function() {
                    var body = mc.deserialize();
                    log.info('Got ping:', body);
                    log.trace('messageContext:', mc);
                    return {
                        value: options.pongValue,
                    };
                })
            });
    });

<!-- /code -->

Now we can create the ping-sender, which will send ping messages to the topic 
exchange and wait for a reply from the listener:

<!-- code lang=javascript -->

    var crutch = require('../crutch.js');
    var defaults = {
        // medseek-util-microservices options
        defaultExchange: 'topic://example',
        defaultQueue: 'ping-sender',
        defaultReturnBody: true, // false returns message-context from .call/etc
    
        // microservice-crutch options
        'log.level': 'ERROR',
    
        // custom options
        //   override on command line like
        //     --pingValue=MYPING
        pingValue: 'PING',
    };
    
    crutch(defaults, function(app, logging, microservices, options, Promise) {
        var log = logging.getLogger('ping-sender');
        log.setLevel('INFO');
    
        // return a promise that is resolved when your application is ready
        return Promise
            .delay(100)
            .then(function() {
                var body = {
                    value: options.pingValue,
                };
                var properties = {
                    'sent-by': 'ping-sender',
                };
                microservices.call('example.ping', body, properties)
                    .then(function(reply) {
                        log.info('Got reply:', reply);
                    })
                    .finally(app.shutdown)
                    .done();
            });
    });

<!-- /code -->

We can now start the ping listener, and the ping sender after that:

<!-- code lang=bash -->

    node ping-listener.js
    node ping-sender.js

<!-- /code -->

You should now see the services exchange PING/PONG messages.

For more information, visit the GitHub project: https://github.com/fwestrom/microservice-crutch.
