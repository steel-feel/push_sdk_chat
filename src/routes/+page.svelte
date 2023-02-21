<script>
    import * as sdk from "@pushprotocol/restapi";
    import { onMount } from "svelte";
    import { createSocketConnection, EVENTS } from "@pushprotocol/socket";

    let details = "";
    //@ts-ignore
    let accounts = [],
        //@ts-ignore
        decryptedPvtKey,
        connectedUser; //IUser

    let message = "",
        receiptient = "",
        fChat = [],
        recUser = "",
        chatHistory = [];

    let userNotFound = false;
    async function start() {
        //fetch the chats, the user is talking to
        accounts = await window.ethereum.request({
            method: "eth_requestAccounts",
        }); // Waits for connection to MetaMask.

        //create a user
        connectedUser = await sdk.user.get({
            env: "prod",
            account: accounts[0],
        });

        window.connectedUser = connectedUser;
        if (connectedUser == null || connectedUser?.publicKey.length == 0) {
            userNotFound = true;
        }
    }
    onMount(start);
    async function sendMessage() {
        //@ts-ignore
        details = await sdk.chat.send({
            account: accounts[0],
            pgpPrivateKey: decryptedPvtKey,
            receiverAddress: receiptient,
            env: "prod",
            messageType: "Text",
            messageContent: message,
        });
    }

    async function fetchChats() {
        chatHistory = await sdk.chat.chats({
            account: accounts[0],
            pgpPrivateKey: decryptedPvtKey,
            toDecrypt: true,
            env: "prod",
        });

        // console.log(chatHistory);
    }

    async function fullchat(rec) {
        recUser = rec;
        const env = "prod";
        const threadhash = await sdk.chat.conversationHash({
            account: accounts[0],
            conversationId: rec, // receiver's address or chatId of a group
            env,
        });

        fChat = await sdk.chat.history({
            threadhash: threadhash.threadHash,
            account: accounts[0],
            pgpPrivateKey: decryptedPvtKey,
            limit: 10,
            toDecrypt: true,
            env,
        });

        console.log(fChat);
    }

    async function connectPush() {
        decryptedPvtKey = await sdk.chat.decryptWithWalletRPCMethod(
            connectedUser.encryptedPrivateKey, //encrypted private key
            accounts[0] //user address
        );

        window.decryptedPvtKey = decryptedPvtKey;
    }

    async function createUser() {
        connectedUser = await sdk.user.create({
            account: accounts[0],
            env: "prod",
        });

        console.log({ connectedUser });
        userNotFound = connectedUser.publicKey.length == 0;
    }

    async function fetchNotifications() {
        const pushSDKSocket = createSocketConnection({
            user: "0xA9E78cef5e6c0081b68AdA2554c04198DfF17C69",
            env: "prod",
            apiKey: "jVPMCRom1B.iDRMswdehJG7NpHDiECIHwYMMv6k2KzkPJscFIDyW8TtSnk4blYnGa8DIkfuacU0",
            socketType: "chat",
            socketOptions: { autoConnect: true, reconnectionAttempts: 3 },
        });

        //@ts-ignore
        pushSDKSocket.on(EVENTS.CHAT_RECEIVED_MESSAGE, async (message) => {
            console.debug({ message });

        });
    }

    async function dMessage() {
        let m = "<Message-to-decrypt>" ;


            const dchat = await sdk.chat.decryptConversation({
                messages: [m], //array of message object fetched from chat.history method
                connectedUser, // user meta data object fetched from chat.get method
                pgpPrivateKey: decryptedPvtKey, //decrypted private key
                env: "prod",
            });

            console.log({ dchat });
    }
</script>

<h1>Push Chat SDK DEMO</h1>
<!-- <p>{JSON.stringify(details)}</p> -->

<button on:click={fetchNotifications}>fetch Notification</button>

<button on:click={dMessage}>dMessage</button>

{#if userNotFound}
    <h2>User not present please create a user</h2>
    <button on:click={createUser}>Create user</button>
{/if}

{#if !userNotFound}
    <button on:click={connectPush} style="margin: 1rem;"
        >Connect Push Chat</button
    >
    <div class="container">
        <input
            placeholder="receipt address"
            style="margin: 0.5rem;"
            bind:value={receiptient}
            type="text"
        />
        <textarea
            placeholder="Message"
            style="margin: 0.5rem;"
            bind:value={message}
        />
        <div class="button-row">
            <button style="width:50%;" on:click={sendMessage}
                >Send Message</button
            >
            <button style="width:50%;" on:click={fetchChats}>Fetch push</button>
        </div>
    </div>

    <div class="container">
        <h3>{accounts[0]} Chat History</h3>
        <ul>
            {#each chatHistory as chat}
                <li>
                    <h5>User <span>{chat.toCAIP10}</span></h5>
                    <p>Last message: <span>{chat.messageContent}</span></p>
                    <button on:click={() => fullchat(chat.toCAIP10)}
                        >fetch full chat</button
                    >
                </li>
            {/each}
        </ul>
    </div>

    <h3>Full chat with {recUser}</h3>

    <ul>
        {#each fChat as chat}
            <li>
                <h5>From <span>{chat.fromCAIP10}</span></h5>
                <h5>Time: <span> {new Date(chat.timestamp)}</span></h5>
                <p>message: <span>{chat.messageContent}</span></p>
            </li>
        {/each}
    </ul>
{/if}

<style>
    .container {
        display: flex;
        flex-direction: column;
        width: 50%;
    }

    .button-row {
        display: flex;
        flex-direction: row;
    }
</style>
