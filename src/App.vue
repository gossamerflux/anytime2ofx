<template>
  <div id="app">
    <h1>This button converts a transaction export from Anytime&nbsp;Bank to an OFX file for import into accounting software like Quickbooks.</h1>
    <h2><a href="https://github.com/gossamerflux/anytime2ofx">Open source</a>. Your data never leaves your computer.</h2>
    <label id="file-input" :class="isDownloading ? 'success' : ''">
      {{ currentStatus }}
      <input type="file" @input="parseCsv($event)" @change="$event.target.value = ''" />
    </label>
    <a href="https://github.com/gossamerflux/anytime2ofx" class="github-corner" aria-label="View source on GitHub"><svg width="80" height="80" viewBox="0 0 250 250" style="fill:#4283b9; color:#fff; position: absolute; top: 0; border: 0; right: 0;" aria-hidden="true"><path d="M0,0 L115,115 L130,115 L142,142 L250,250 L250,0 Z"></path><path d="M128.3,109.0 C113.8,99.7 119.0,89.6 119.0,89.6 C122.0,82.7 120.5,78.6 120.5,78.6 C119.2,72.0 123.4,76.3 123.4,76.3 C127.3,80.9 125.5,87.3 125.5,87.3 C122.9,97.6 130.6,101.9 134.4,103.2" fill="currentColor" style="transform-origin: 130px 106px;" class="octo-arm"></path><path d="M115.0,115.0 C114.9,115.1 118.7,116.5 119.8,115.4 L133.7,101.6 C136.9,99.2 139.9,98.4 142.2,98.6 C133.8,88.0 127.5,74.4 143.8,58.0 C148.5,53.4 154.0,51.2 159.7,51.0 C160.3,49.4 163.2,43.6 171.4,40.1 C171.4,40.1 176.1,42.5 178.8,56.2 C183.1,58.6 187.2,61.8 190.9,65.4 C194.5,69.0 197.7,73.2 200.1,77.6 C213.8,80.2 216.3,84.9 216.3,84.9 C212.7,93.1 206.9,96.0 205.4,96.6 C205.1,102.4 203.0,107.8 198.3,112.5 C181.9,128.9 168.3,122.5 157.7,114.1 C157.9,116.9 156.7,120.9 152.7,124.9 L141.0,136.5 C139.8,137.7 141.6,141.9 141.8,141.8 Z" fill="currentColor" class="octo-body"></path></svg></a>
  </div>
</template>

<script>
import Vue from 'vue'
import ofx from '@wademason/ofx'
import VuePapaParse from 'vue-papa-parse'
import VueMoment from 'vue-moment'
Vue.use(VuePapaParse)
Vue.use(VueMoment)


export default {
  name: 'App',

  data() {
    return {
      isDownloading: false,
      filename: "transactions-" + this.formatDatetime(),
      transactions: []
    }
  },

  methods: {
    downloadOfx() {
     try {
        let ofxData = new Blob([this.ofxString], { type: 'application/x-ofx' })
        let ofxUrl = URL.createObjectURL(ofxData)

        let link = document.createElement('a')
        link.id = 'ofx-' + parseInt(Math.random().toString().slice(2,16))
        link.href = ofxUrl

        document.body.appendChild(link)

        let $el = document.getElementById(link.id)

        $el.style.visibility = 'hidden'
        $el.download = this.filename + '.ofx'
        $el.click()

        setTimeout(function () {
          document.body.removeChild(link)
        })

        return true
      } catch (err) {
        return false
      }
    },

    parseCsv(event) {
      if(event.target.files[0]) {
        this.filename = event.target.files[0].name.split(".")[0]
        this.$papa.parse(event.target.files[0], { header: true, delimiter: ";", encoding: "windows-1252", complete: this.processCsv })
      }
    },

    processCsv(results) {
      this.transactions = []
      if(results && results.data) {
        let data = results.data
        
        for(let row of data) {
          if(!row['Date'] || (!row['Montant'] && !row['Amount'])) { continue }
          this.transactions.push({
            timestamp: row['Date'],
            reference: row['Référence'] || row['Reference'] || this.generateHash(row),
            description: row['Description'] || "",
            amount: row['Montant'] || row['Amount'],
          })
        }
      }

      this.$nextTick(() => {
        this.isDownloading = true
        setTimeout(() => this.isDownloading = false, 2000)
        this.downloadOfx()
      })
    },

    formatDatetime(date) {
      return this.$moment(date).format('YYYYMMDDhhmmss')
    },

    generateHash(obj) {
      var str = JSON.stringify(obj)
      var hash = 0, i, chr;
      for (i = 0; i < str.length; i++) {
        chr   = str.charCodeAt(i);
        hash  = ((hash << 5) - hash) + chr;
        hash |= 0; // Convert to 32bit integer
      }
      return Math.abs(hash);
    }
  },

  computed: {

    currentStatus() {
      return this.isDownloading ? "Your OFX file is downloading" : "Select an Anytime CSV file"
    },

    ofxString() {
      return ofx.serialize(this.ofxHeaders, this.ofxBody)
    },

    ofxHeaders() {
      return {
        "OFXHEADER":"100",
        "DATA":"OFXSGML",
        "VERSION":"102",
        "SECURITY":"NONE",
        "ENCODING":"USASCII",
        "CHARSET":"1252",
        "COMPRESSION":"NONE",
        "OLDFILEUID":"NONE",
        "NEWFILEUID":"NONE"
      }
    },
    
    ofxBody() {
      return {
        "OFX":{
          "SIGNONMSGSRSV1":{
            "SONRS":{
              "STATUS":{
                "CODE":{
                  "_text":"0"
                },
                "SEVERITY":{
                  "_text":"INFO"
                }
              },
              "DTSERVER":{
                "_text": this.formatDatetime()
              },
              "LANGUAGE":{
                "_text":"FRA"
              }
            }
          },
          "BANKMSGSRSV1":{
            "STMTTRNRS":{
              "TRNUID":{
                "_text":"00000000000"
              },
              "STATUS":{
                "CODE":{
                  "_text":"0"
                },
                "SEVERITY":{
                  "_text":"INFO"
                }
              },
              "STMTRS":{
                "CURDEF":{
                  "_text":"EUR"
                },
                "BANKACCTFROM":{
                  "BANKID":{
                    "_text":"00000"
                  },
                  "BRANCHID":{
                    "_text":"00000"
                  },
                  "ACCTID":{
                    "_text":"00000000000"
                  },
                  "ACCTTYPE":{
                    "_text":"CHECKING"
                  }
                },
                "BANKTRANLIST":{
                  "DTSTART":{
                    "_text": this.formattedFirstTransactionDatetime, 
                  },
                  "DTEND":{
                    "_text": this.formattedLastTransactionDatetime, 
                  },
                  "STMTTRN": this.formattedTransactions
                },
                "LEDGERBAL":{
                  "BALAMT":{
                    "_text":""
                  },
                  "DTASOF":{
                    "_text": this.formattedLastTransactionDatetime
                  }
                },
                "AVAILBAL":{
                  "BALAMT":{
                    "_text":""
                  },
                  "DTASOF":{
                    "_text": this.formattedLastTransactionDatetime
                  }
                }
              }
            }
          }
        }      
      }
    },

    formattedFirstTransactionDatetime() {
      let firstTransaction = this.sortedTransactions[0]
      return firstTransaction ? this.formatDatetime(firstTransaction.timestamp) : null
    },

    formattedLastTransactionDatetime() {
      let lastTransaction = this.sortedTransactions[this.sortedTransactions.length - 1]
      return lastTransaction ? this.formatDatetime(lastTransaction.timestamp) : null
    },

    sortedTransactions() {
      return this.transactions.slice().sort(function(a, b) {
        let comparison = 0;
        if (a.timestamp > b.timestamp) {
          comparison = 1
        } else if (a.timestamp < b.timestamp) {
          comparison = -1
        }
        return comparison
      })
    },

    formattedTransactions() {
      return this.sortedTransactions.map(trx => {
        return {
          "TRNTYPE":{
            "_text": "OTHER"
          },
          "DTPOSTED":{
            "_text": this.$moment(trx.timestamp).format("YYYYMMDD")
          },
          "TRNAMT":{
            "_text": trx.amount.replace(",", ".")
          },
          "FITID":{
            "_text": trx.reference
          },
          "NAME":{
            "_text": trx.description
          },
          "MEMO":{
            "_text": "."
          }
        }
      })
    }
  }
}
</script>

<style lang="css">
body {
  padding: 0;
  margin: 0;
  background-color: #FCFCFC;
}
#app {
  flex-direction: column;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
h1, h2 {
  font-family: Helvetica, Arial, sans-serif;
  font-weight: 400;
  text-align: center;
  max-width: 36vw;
  line-height: 150%;
}
h1 {
  font-size: calc(1vw + 0.5em);
  color: #555;
  margin-bottom: 0.25em;
}
h2 {
  font-size: calc(0.8vw + 0.4em);
  color: #666;
  margin-bottom: 2em;
}
a, a:active, a:focus, a:visited {
  text-decoration: none;
  color: #4283b9;
}
#file-input {
  font-family: Helvetica, Arial, sans-serif;
  font-weight: 900;
  font-size: calc(1.8vw + 0.5em);
  min-width: calc(28vw + 2em);
  text-align: center;
  border-radius: 10vw;
  padding: calc(0.8vw + 0.25em) calc(1.6vw + 0.5em);
  letter-spacing: -0.01vw;
  background-color: #4283b9;
  color: white;
  cursor: pointer !important;
  transition: background-color 0.2s ease-in-out;
  margin-bottom: 1em;
}

#file-input.success {
  background-color: #42b983;
}
  
#file-input > input {
  position: absolute;
  z-index: -1;
  visibility: hidden;
}

.github-corner:hover .octo-arm{animation:octocat-wave 560ms ease-in-out}@keyframes octocat-wave{0%,100%{transform:rotate(0)}20%,60%{transform:rotate(-25deg)}40%,80%{transform:rotate(10deg)}}@media (max-width:500px){.github-corner:hover .octo-arm{animation:none}.github-corner .octo-arm{animation:octocat-wave 560ms ease-in-out}}
</style>
