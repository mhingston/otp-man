<template>
  <q-page padding class="row justify-center">
    <div style="width: 100%">
      <q-list link
        v-for="entry in otp"
        :key="entry.id"
      >
        <q-item @click.native="(event) => showOTP(event, entry)">
          <q-item-side left>
            <input class="otpcode" type="text" :value="entry.code" readonly="readonly">
          </q-item-side>
          <q-item-main>
            <q-item-tile label>{{entry.label}}</q-item-tile>
            <q-item-tile sublabel>{{entry.issuer}}</q-item-tile>
          </q-item-main>
          <q-item-side right>
            <q-knob
              :min="entry.min"
              :max="entry.max"
              color="primary"
              :value="entry.timeRemaining"
              readonly
            />
            <q-btn @click.native="(event) => removeOTP(event, entry)" round color="secondary" icon="delete" />
          </q-item-side>
        </q-item>
      </q-list>
      <q-page-sticky position="bottom-right" :offset="[18, 18]">
        <q-btn
          round
          color="primary"
          @click="selectFile"
          icon="add"
          class="q-btn-fab"
          :disabled="password === null"
        >
          <q-tooltip
            anchor="center right"
            self="center left"
          >
            Add OTP QR code
          </q-tooltip>
        </q-btn>
        <input type="file" v-on:change="addOTP">
      </q-page-sticky>
    </div>
  </q-page>
</template>

<script>
import * as jsQR from 'jsqr'
import * as queryString from 'query-string'
import * as speakeasy from 'speakeasy'
import * as CryptoJS from 'crypto-js'
export default {
  name: 'PageIndex',
  data: () =>
  ({
    addProps (entry)
    {
      entry.min = 0,
      entry.max = parseInt(entry.period)
      entry.timeRemaining = null,
      entry.code = new Array(parseInt(entry.digits)).join('-')
    },
    save ()
    {
      try
      {
        const salt = this.$q.localStorage.get.item('salt')
        const key = CryptoJS.PBKDF2(this.password, salt,
        {
          keySize: 512 / 32,
          iterations: 1000
        }).toString()

        const cipherText = CryptoJS.AES.encrypt(JSON.stringify(this.otp), key).toString()
        this.$q.localStorage.set('otp', cipherText)
      }

      catch(error)
      {
        this.$q.notify(
        {
          type: 'negative',
          message: 'Unable to encrypt OTP codes.'
        })
      }
    },
    otp: [],
    password: null
  }),
  methods: {
    selectFile()
    {
      this.$el.querySelector('input[type="file"]').click()
    },
    showOTP(event, otp)
    {
      let options
      switch(otp.type)
      {
        case 'totp':
          options =
          {
            secret: otp.secret,
            time: otp.time,
            step: otp.period,
            epoch: otp.epoch,
            counter: otp.counter,
            digits: otp.digits,
            encoding: 'base32',
            algorithm: otp.algorithm
          }
          otp.code = options.token = speakeasy.totp(options)
          otp.timeRemaining = Math.round(speakeasy.totp.verifyDelta(options).timeRemaining / 1000)
          break

        case 'hotp':
          options =
          {
            secret: otp.secret,
            counter: otp.counter,
            digits: otp.digits,
            encoding: 'base32',
            algorithm: otp.algorithm
          }
          otp.code = options.token = speakeasy.hotp(options)
          otp.timeRemaining = 30
          break
      }

      clearInterval(otp.timer)
      otp.timer = setInterval(() =>
      {
        otp.timeRemaining--;

        if(otp.timeRemaining < 1)
        {
          this.addProps(otp)
          clearInterval(otp.timer)
        }
      }, 1000)

      let target = event.target

      while(target.className !== 'q-item q-item-division relative-position')
      {
        target = target.parentNode
      }

      const otpCode = target.querySelector('.otpcode')
      target.querySelector('.otpcode').value = otp.code
      otpCode.select()
      document.execCommand('copy')
      otpCode.blur()
      window.getSelection().removeAllRanges()
      this.$q.notify(
      {
        type: 'positive',
        message: 'OTP code copied to clipboard.'
      })
    },
    addOTP(event)
    {
      const file = event.target && event.target.files ? event.target.files[0] : null

      if(event.target.files[0])
      {
        const reader = new FileReader()

        reader.addEventListener('load', () =>
        {
          const img = new Image()
          img.onload = () =>
          {
            const canvas = document.createElement('canvas')
            canvas.width = img.width
            canvas.height = img.height
            const ctx = canvas.getContext('2d')
            ctx.drawImage(img, 0, 0, img.width, img.height)
            const imageData = ctx.getImageData(0, 0, img.width, img.height)
            const code = jsQR(imageData.data, img.width, img.height)
            const matches = code.data.match(/otpauth:\/\/(\w+)\/([^?]+)(\?.*)/)

            if(matches)
            {
              const type = matches[1]
              const label = matches[2]
              let otp = queryString.parse(matches[3])
              otp = Object.assign(otp, {type, label})
              this.addProps(otp)
              otp.id = this.otp.length ? this.otp[this.otp.length-1].id+1 : 0
              this.otp.push(otp)
              this.save()
            }
          }
          img.src = reader.result
        })

        reader.readAsDataURL(file)
      }
    },
    removeOTP(event, otp)
    {
      for(let i=0; i < this.otp.length; i++)
      {
        if(this.otp[i].id === otp.id)
        {
          this.otp.splice(i, 1)
          break
        }
      }

      this.save()
      event.stopPropagation()
    }
  },
  mounted: async function()
  {
    let existingData = this.$q.localStorage.get.item('otp') ? true : false

    const password = await this.$q.dialog(
    {
      title: 'Password Required',
      message: existingData ? 'Please enter your password.' : 'Please enter a password to secure your OTP codes.',
      prompt: {
        type: 'password'
      },
      preventClose: true
    })

    if(!existingData)
    {
      const salt = CryptoJS.lib.WordArray.random(128 / 8).toString()
      this.$q.localStorage.set('salt', salt)
      return
    }

    try
    {
      const salt = this.$q.localStorage.get.item('salt')
      const key = CryptoJS.PBKDF2(password, salt,
      {
        keySize: 512 / 32,
        iterations: 1000
      }).toString()

      const cipherText = this.$q.localStorage.get.item('otp')
      const plainText = CryptoJS.AES.decrypt(cipherText, key).toString(CryptoJS.enc.Utf8)
      const otp = JSON.parse(plainText)
      otp.forEach(this.addProps)
      this.otp = otp
      this.password = password
    }

    catch(error)
    {
      this.$q.notify(
      {
        type: 'negative',
        message: 'Unable to decrypt OTP codes.'
      })
    }
  }
}
</script>

<style>
input[type='file'] {
  display: none;
}

input.otpcode {
  width: 200px;
  cursor: pointer;
  font-size: 24pt;
  border: 0;
  background: none;
  outline: 0;
  color: transparent;
  text-shadow: 0 0 0 #2196f3;
}
</style>
