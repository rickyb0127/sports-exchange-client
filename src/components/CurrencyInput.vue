<template>
  <div>
    <div class="prefix inline">$</div>
    <div class="inline">
      <md-input
        ref="currencyInput"
        v-model="localCurrencyToFormat"
        @input="handleInput()"
        @click="selectAll()"
        @blur="sanitizeCurrency()"
        @keypress="formatCurrencyForInput($event)"
        @focus="convertToString()">
      </md-input>
    </div>
  </div>
</template>
<script>

export default {
  name: 'CurrencyInput',
  data() {
    return {
      isNegative: false,
      attrs: {
        localCurrencyToFormat: this.currencyToFormat
      }
    }
  },
  created() {
    this.sanitizeCurrency();
  },
  computed: {
    localCurrencyToFormat: {
      get() {
        return this.attrs.localCurrencyToFormat;
      },
      set(value) {
        this.attrs.localCurrencyToFormat = value;
        if(this.isNegative) {
          value = `-${value}`;
        }
        if(value) {
          if(value.indexOf(',') > -1 || value.indexOf('(') > -1 || value.indexOf(')') > -1) {
            value = value.replace(/,/g, '').replace('(', '').replace(')', '');
          }
        }
        this.$emit(`update:currency-to-format`, parseFloat(value));
      }
    }
  },
  props: {
    currencyToFormat: {
      type: [Number, String]
    },
    allowsNegative: {
      type: Boolean
    },
    inputCallback: {
      type: Function
    },
    blurCallback: {
      type: Function
    },
    allowsNullVal: {
      type: Boolean
    }
  },
  methods: {
    selectAll() {
      const input = this.$refs.currencyInput.$el;
      input.focus();
      input.select();
      this.removeCommas();
    },
    sanitizeCurrency() {
      this.convertToString();
      
      if(!this.localCurrencyToFormat) {
        if(!this.allowsNullVal) {
          this.localCurrencyToFormat = "0.00";
        } else {
          return;
        }
      }
      this.isNegative = false;
      if(this.localCurrencyToFormat.indexOf('-') > -1 || this.localCurrencyToFormat.indexOf('(') > -1 || this.localCurrencyToFormat.indexOf(')') > -1){
        const noZeros = this.localCurrencyToFormat.replace(/0/g, '');
        const digitRegex = /[0-9]/g;
        if(digitRegex.test(noZeros)) {
          if(this.localCurrencyToFormat.indexOf('-') === 0 || this.localCurrencyToFormat.indexOf('(') === 0) {
            this.isNegative = true;
          }
        }
        this.localCurrencyToFormat = this.localCurrencyToFormat.replace(/-/g, '').replace('(', '').replace(')', '');
      }
      while(this.localCurrencyToFormat.indexOf('0') === 0) {
        this.localCurrencyToFormat = this.localCurrencyToFormat.substring(1);
      }
      if(this.localCurrencyToFormat) {
        if(this.localCurrencyToFormat.indexOf('.') === 0) {
          this.localCurrencyToFormat = '0' + this.localCurrencyToFormat;
        }
      } else {
        this.localCurrencyToFormat = '0';
      }
      if(this.localCurrencyToFormat.indexOf('.') > -1) {
        let decimalIndex = this.localCurrencyToFormat.indexOf('.');
        let trailingDecimalStr = this.localCurrencyToFormat.substring(decimalIndex + 1);
        if(trailingDecimalStr.length > 2) {
          this.localCurrencyToFormat = this.localCurrencyToFormat.substring(0, decimalIndex + 3);
        }
        if(trailingDecimalStr.length === 0) {
          this.localCurrencyToFormat += '00';
        }
        if(trailingDecimalStr.length === 1) {
          this.localCurrencyToFormat += '0';
        }
      } else {
        this.localCurrencyToFormat += '.00';
      }
      if(this.isNegative) {
        this.localCurrencyToFormat = `(${this.localCurrencyToFormat})`;
      }
      this.addCommas(this.localCurrencyToFormat);
      let strippedCurrency = this.localCurrencyToFormat.replace(/,/g, '').replace('(', '').replace(')', '');
      if(isNaN(parseFloat(strippedCurrency))) {
        this.localCurrencyToFormat = "";
      }
      if(this.blurCallback) {
        this.blurCallback();
      }
    },
    formatCurrencyForInput(event) {
      if(!this.allowsNegative) {
        let keyCode = (event.keyCode ? event.keyCode : event.which);
        if((keyCode < 48 || keyCode > 57) 
          && (keyCode !== 46 || this.localCurrencyToFormat.indexOf('.') != -1)) {
            event.preventDefault();
          }
      }
      if(this.localCurrencyToFormat) {
        let keyCode = (event.keyCode ? event.keyCode : event.which);
        if(this.allowsNegative) {
          if((keyCode < 48 || keyCode > 57) 
          && (keyCode !== 46 || this.localCurrencyToFormat.indexOf('.') != -1) 
          && (keyCode !== 45 || this.localCurrencyToFormat.indexOf('-') != -1) 
          && (keyCode !== 40 || this.localCurrencyToFormat.indexOf('(') != -1) 
          && (keyCode !== 41 || this.localCurrencyToFormat.indexOf(')') != -1)) {
            event.preventDefault();
          }
        } else {
          if((keyCode < 48 || keyCode > 57) 
          && (keyCode !== 46 || this.localCurrencyToFormat.indexOf('.') != -1)) {
            event.preventDefault();
          }
        }
      }
    },
    addCommas() {
      let str = this.localCurrencyToFormat.toString().split('.');
      if(str[0].length >= 4) {
        str[0] = str[0].replace(/(\d)(?=(\d{3})+$)/g, '$1,');
      }
      if(str[1] && str[1].length >= 5) {
        str[1] = str[1].replace(/(\d{3})/g, '$1 ');
      }
      this.localCurrencyToFormat = str.join('.');
    },
    removeCommas() {
      if(this.localCurrencyToFormat && this.localCurrencyToFormat.includes(",")) {
        this.localCurrencyToFormat = this.localCurrencyToFormat.replace(/,/g, '');
      }
    },
    convertToString() {
      if(typeof this.localCurrencyToFormat === "number") {
        this.localCurrencyToFormat = this.localCurrencyToFormat.toString();
      }
    },
    handleInput() {
      if(this.inputCallback) {
        this.inputCallback();
      }
    }
  }
}
</script>

<style scoped>
  .prefix {
    font-size: 16px;
    line-height: 32px;
    margin-right: 5px;
  }
  .inline {
    display: inline-block;
  }
</style>