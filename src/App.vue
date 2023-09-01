<template>
    <div class="ecp-wrapper">
        <div class="ecp-card">
            <h2>Цифровий підпис</h2>
            <iframe :src="url"
                    loading="lazy"
                    :ref="id"
                    referrerpolicy="no-referrer"
                    @load="loading = false"></iframe>
            <div class="ecp-btns">
                <button v-if="loaded" @click="sign">Підписати</button>
                <button>Відмінити</button>
            </div>
        </div>
    </div>
</template>
<script>
export default {
    data() {
        return {
            loading: true,
            loaded: false,
            s_debug: false,
            sender: "EndUserSignWidgetConnector",
            reciever: "EndUserSignWidget",
            version: "20200922",
            parentId: "sign-widget-parent",
            id: "sign-widget",
            src: "https://eu.iit.com.ua/sign-widget/v20200922/",
            formType: 0,
            formParams: null,
            iframe: null,
            m_promises: [],
            FormType: {
                "ReadPKey":				1,
                "MakeNewCertificate":	2,
                "SignFile":				3,
                "ViewPKeyCertificates":	4
            },
            SignAlgo: {
                "DSTU4145WithGOST34311":	1,
                "RSAWithSHA":				2,
                "ECDSAWithSHA":				3
            },
            SignType: {
                "CAdES_BES":			1,
                "CAdES_T":				4,
                "CAdES_C":				8,
                "CAdES_X_Long":			16
            },

        };
    },
    created() {
        this.formType = this.FormType.ReadPKey;
    },
    mounted() {
        this.iframe = this.$refs[this.id];
        window.addEventListener("message", this.listener, false);
        this.load();
    },
    computed: {
        url() {
            let srcParams = '?address=' + this._parseURL(window.location.href).origin;
            srcParams += '&formType=' + this.formType;
            srcParams += '&debug=' + true;
            if (this.formParams) {
                for (let paramName in this.formParams)
                    srcParams += '&' + paramName + '=' + this.formParams[paramName];
            }
            return this.src + srcParams;
        },
    },
    methods: {
        listener(event) {
            const origin = this._parseURL(this.src).origin;
            if(event.origin !== origin) return;
            this._recieveMessage(event);
        },
        load() {
            this.ReadPrivateKey().then((res) => {
                console.log(res)
                this.loaded = true;
            }).catch((e) => {
                console.log('Виникла помилка при зчитуванні особистого ключа: ' + (e.message || e))
            });
        },
        sign() {
            const data = this.value instanceof Int8Array ? this.value : JSON.stringify(this.value);
            const previousSign = null;
            const external = false;
            const asBase64String = true;
            const signAlgo = this.SignAlgo.DSTU4145WithGOST34311;
            const signType = this.SignType.CAdES_X_Long;
            this.SignData(data, external, asBase64String, signAlgo, previousSign, signType)
                .then((sign) => {
                    this.$emit('sign', sign);
                    this.show = false;
                })
                .catch((e) => {
                    console.log('Виникла помилка при підписуванні даних: ' + (e.message || e))
                });
        },
        _parseURL(url) {
            const urlRegEx = new RegExp([
                '^(https?:)//',
                '(([^:/?#]*)(?::([0-9]+))?)',
                '(/{0,1}[^?]*)'
            ].join(''));
            const match = url.match(urlRegEx);
            return match && {
                protocol: match[1],
                host: match[2],
                hostname: match[3],
                port: match[4],
                pathname: match[5],
                origin: match[1] + '//' + match[2]
            };
        },
        _postMessage(cmd, params, _resolve, _reject) {
            let p = null;
            const msg = {
                sender: this.sender,
                reciever: this.reciever,
                id: -1,
                cmd: cmd,
                params: params
            };

            if (typeof _resolve == 'undefined' && typeof _reject == 'undefined') {
                p = new Promise((resolve, reject) => {
                    msg.id = this.m_promises.push({
                        resolve: resolve,
                        reject: reject,
                        msg: msg
                    });
                });
            } else {
                msg.id = this.m_promises.push({
                    resolve: _resolve,
                    reject: _reject,
                    msg: msg
                });
            }

            try {
                this.iframe.contentWindow.postMessage(msg, this._parseURL(this.src).origin);
            } catch (e) {
                if(this.s_debug)
                    console.log("Main page post message error: " + e);
            }

            if(this.s_debug)
                console.log("Main page post message: " + msg);

            return p;
        },
        _recieveMessage(event) {
            if (this.s_debug)
                console.log("Main page recieve message: " + event.data);

            const data = event.data;
            if (data.reciever !== this.sender || data.sender !== this.reciever) {
                return;
            }

            if (data.id === -1) {
                const promises = this.m_promises;
                this.m_promises = [];
                for(let i = 0; i < promises.length; i++) {
                    this._postMessage(
                        promises[i].msg.cmd, promises[i].msg.params,
                        promises[i].resolve, promises[i].reject);
                }
                return;
            }

            const p = this.m_promises[data.id - 1];
            if(!p) return;
            delete this.m_promises[data.id - 1];
            data.error ? p.reject(data.error) : p.resolve(data.result);
        },
        ReadPrivateKey() {
            const params = Array.prototype.slice.call(arguments);
            return this._postMessage('ReadPrivateKey', params);
        },
        SignData(data, external, asBase64String, signAlgo, previousSign, signType) {
            const params = Array.prototype.slice.call(arguments);
            return this._postMessage('SignData', params);
        },
    },
}
</script>
<style>
body {
    margin: 0;
    font-family: 'Roboto';
}
.ecp-wrapper  {
    display: flex;
    height: 100vh;
    width: 100%;
    align-items: center;
    justify-content: center;
    background: rgba(0, 0, 0, 0.45);
}
.ecp-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    background: white;
    border-radius: 8px;
}
.ecp-btns {
    margin: 24px 0;
    display: flex;
    gap: 16px;
}
iframe {
    width: 600px;
    min-height: 450px;
}
h2 {
    color: rgba(0, 0, 0, 0.87);
    background: #eff2e9;
    text-align: center;
    margin: 0;
    width: 100%;
    line-height: 60px;
}
button {
    max-width: 150px;
    padding: 10px 30px;
    background: #26ae4d;
    color: #ffffff;
    border-radius: 0.3rem;
}
button:first-child {
    border: none;
    background: #26ae4d;
    color: #ffffff;
}
button:last-child {
    border: 1px solid #26ae4d;
    background: transparent;
    color: #000000;
}
button:hover {
    box-shadow: 0 6px 10px rgba(0, 0, 0, 0.2);
    cursor: pointer
}
button:first-child:hover {
    background: #0fa039;
}
</style>


