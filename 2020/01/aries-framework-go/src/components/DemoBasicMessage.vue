<template >
    <div class="eg-transition" :enter='enter' :leave='leave'>
    <div class="eg-slide" v-if="active">
    <div class="eg-slide-content">
        <h2>Basic Message</h2>
        <div v-if="step === 1">
            Get an invitation from the bot -
            <button class="button" v-on:click="getBotInvitation">Retrieve</button>
            <eg-transition enter='fadeIn' leave='bounceOutLeft' style=padding-top:10px>
                <eg-code-block lang="json" v-if="invitation != null && step === 1" enter='flipInY'>{{ invitation }}</eg-code-block>
            </eg-transition>
        </div>

        <div v-if="step < 5">
            <p v-if="invitation != null && step > 1">{{ invitation.label }} endpoint: <b>{{ invitation.serviceEndpoint }}</b></p>
        </div>

        <div v-if="step === 2">Connect to the bot -
            <button class="button" v-on:click="connectToBot">Connect</button>
            <eg-transition enter='fadeIn' leave='fadeOut'>
                <p v-if="connectionStatus != null && step === 2">{{ connectionStatus }}</p>
            </eg-transition>
        </div>

        <div v-if="step === 3">Check the connection status with the bot  -
            <button class="button" v-on:click="botConnStatus">Status</button>
            <button class="button" v-on:click="clearBotConnStatus">Clear</button>
            <eg-transition enter='fadeIn'>
                <p v-if="botConnnectionStatus != null && step === 3">{{ botConnnectionStatus }}</p>
            </eg-transition>
        </div>

        <div v-if="step === 4">Register for messages -
            <button class="button" v-on:click="humanRegistration">Register</button>
            <p v-if="registerStatus != null && step === 4">{{ registerStatus }}</p>
        </div>

        <div v-if="step === 5 || step === 6 || step === 7">Chat with the bot -</div>
        <div v-if="step === 5">
            <div class="center">
                <div>What kind of pie do you like?</div>
                <input v-model='pie' />
                <button class="button" v-on:click="sendMessage">Send</button>
            </div>
        </div>
        <div v-if="step === 6">
            <eg-transition enter='fadeIn' leave='bounceOutLeft' style=padding-top:10px>
                <eg-code-block lang="json" v-if="botMessageResponse != null && (step === 5 || step === 6)" enter='flipInY'>{{ botMessageResponse }}</eg-code-block>
            </eg-transition>
        </div>
        <div v-if="step === 7">
            <eg-transition enter='fadeIn' leave='bounceOutLeft' style=padding-top:10px>
                <eg-code-block v-if="botMessageContent != null">{{ botMessageContent }}</eg-code-block>
            </eg-transition>
        </div>
    </div>
    </div>
    </div>
</template>

<script>
    import { AriesREST, AriesWebHook } from '../ariesrest.js'
    import { Slide } from 'eagle.js'
    const uuidv4 = require('uuid/v4');

    export default {
        name: 'DemoBasicMessage',
        props: {
            steps: { default: 7 },
            botAgentURL: String,
            humanAgentURL: String,
            humanAgentWebhookURL: String,
            autoBotMsgRegister: Boolean
        },
        mixins: [Slide],
        data() {
            return {
                pie: "strawberry",
                invitation: null,
                connectionStatus: null,
                connectionID: null,
                registerStatus: null,
                botConnnectionStatus: null,
                botMessageResponse: null,
                botMessageContent: null,
                ariesBotAgentClient: new AriesREST(this.botAgentURL),
                ariesHumanAgentClient: new AriesREST(this.humanAgentURL),
                ariesHumanWebHookClient: new AriesWebHook(this.humanAgentWebhookURL),
            };
        },
        metaInfo: {
            title: 'Basic Message via DIDComm'
        },
        methods: {
            getBotInvitation: function() {
                this.ariesBotAgentClient.didexchange.CreateInvitation({}).then(res => {
                        this.invitation = res.data.invitation;
                    })
            },
            connectToBot: async function() {
                if (this.invitation === null) {
                    this.connectionStatus = "Retrieve the invitation before proceeding"
                } else {
                    this.connectionStatus = "connecting"

                    try {
                        let res = await this.ariesHumanAgentClient.didexchange.ReceiveInvitation(this.invitation)
                        this.connectionID = res.data.connection_id
                    } catch (err) {
                        this.connectionStatus = err
                        return
                    }
                    
                    const attempts = 40
                    for (let i =0; i < attempts; i++) {
                        await new Promise(r => setTimeout(r, 250));
                        let res = await this.ariesHumanAgentClient.didexchange.QueryConnectionByID(this.connectionID)

                        if (res.data.result.State == 'completed') {
                            this.connectionStatus = res.data.result.State
                            break;
                        }

                        if (i == attempts - 1) {
                            this.connectionStatus = "timed out at state: " + res.data.result.State
                        }
                    }
                }
            },
            botRegistration: async function() {
                try {
                    let res = await this.ariesBotAgentClient.messaging.RegisterMessageService({
                            "name" : "basic-message",
                            "type" : "https://didcomm.org/basicmessage/1.0/message"
                        })

                    console.log(res)
                } catch(err) {
                    console.log(err)
                }
            },
            humanRegistration: async function() {
              await this.unregisterMessage(this.ariesHumanAgentClient)
              await this.registerMessage(this.ariesHumanAgentClient)
            },
            registerMessage: async function(agentClient) {
                try {
                    let res = await agentClient.messaging.RegisterMessageService({
                            "name" : "basic-message",
                            "type" : "https://didcomm.org/basicmessage/1.0/message"
                        })

                    console.log(res)
                    this.registerStatus = "success"
                } catch(err) {
                        this.registerStatus = err
                }
            },
            unregisterMessage: async function(agentClient) {
                try {
                    let res = await agentClient.messaging.UnregisterMessageService({
                            "name" : "basic-message",
                        })
                        console.log(res)
                } catch(err) {
                    console.log(err)
                }
            },
            botConnStatus: async function() {
                try {
                    let res = await this.ariesHumanAgentClient.didexchange.QueryConnectionByID(this.connectionID)

                    if (res.data.result.State == 'completed') {
                        this.botConnnectionStatus = "success"
                    } else {
                        this.botConnnectionStatus = "in-progress"
                    }
                } catch(err) {
                    this.botConnnectionStatus = err
                }
            },
            clearBotConnStatus : function() {
                this.botConnnectionStatus = ""
            },
            sendMessage: async function() {
                try {
                    await this.ariesHumanAgentClient.messaging.SendNewMessage({
                            "connection_ID": this.connectionID,
                            "message_body": {
                                "@id" : uuidv4(),
                                "@type" : "https://didcomm.org/basicmessage/1.0/message",
                                "content": "I love " + this.pie + " pie",
                                "sent_time": new Date().toISOString(),
                                "~l10n": {"locale":"en"}
                            },
                        })
                } catch(err) {
                    this.botMessageResponse = err
                }

                this.nextStep()
            },
            refreshTopicsData: async function() {
                try {
                    var topicsResponse = await this.ariesHumanWebHookClient.topics.Check();
                    if (topicsResponse.data.message && topicsResponse.data.message['@type'] === "https://didcomm.org/basicmessage/1.0/message") {
                        this.botMessageResponse = topicsResponse.data;
                        this.botMessageContent = topicsResponse.data.message.content;
                    }
                } catch(err) {
                    this.botMessageResponse = err;
                }
           },
        },
        mounted: function () {
            if (this.autoBotMsgRegister) {
                this.botRegistration();
            }
            this.refreshTopicsData();

            setInterval(function () {
                this.refreshTopicsData();
            }.bind(this), 1000);
        }
  }

</script>

<style>
    .button {
        background-color: #a1f1a1; /* Green */
    }
</style>