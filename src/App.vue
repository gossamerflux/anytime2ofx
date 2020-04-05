<template>
  <div id="app">
    <h1>This button converts a transaction export from Anytime&nbsp;Bank to an OFX file for import into accounting software like Quickbooks.</h1>
    <h2>Your data never leaves your computer.</h2>
    <label id="file-input" :class="isDownloading ? 'success' : ''">
      {{ currentStatus }}
      <input type="file" @input="parseCsv($event)" @change="$event.target.value = ''" />
    </label>
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
        this.$papa.parse(event.target.files[0], { delimiter: ";", complete: this.processCsv })
      }
    },

    processCsv(results) {
      this.transactions = []
      if(results && results.data) {
        let data = results.data
        // Discard header row
        data.shift() 
        
        for(let row of data) {
          if(!row[0] /* date */ || !row[3] /* amount */) { continue }
          this.transactions.push({
            timestamp: row[0],
            reference: row[1],
            description: row[2],
            amount: row[3]
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
  max-width: 40vw;
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
</style>
