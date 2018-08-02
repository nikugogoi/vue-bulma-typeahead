<template>
  <span class="vbta group-login is-marginless" :class="{'vbta-suggest': matches.length && !selected}">
    <input :class="['inputMaterial', 'vbta-hint', { visible: matches.length && !selected }]" type="text" :value="hint" readonly>
    <input 
      v-model="query" 
      required
      class="inputMaterial vbta-input" 
      type="text"
      @focus="$emit('hocus')"
      @keyup.delete="handleDelete($event)"
      @keydown.down.prevent="handleKeyDown($event)" 
      @keydown.up.prevent="handleKeyUp" 
      @keyup.enter.prevent.submit="emitSelect(matches[preselected])"
      :class="{'is-danger' : error}"
      @focusout="$emit('outFocus')"
    >
    <div :class="['vbta-menu', { visible: matches.length && !selected }]" class="suggestion-box"> 
        <ul>
          <li v-for="match in matches" class="vbta-suggestion" @click="emitSelect(match)">
            <figure ref="imageContainer" class="image is-64x64">
              <img :src="match.img_url" />
            </figure>
            <p class="has-text-centered" v-html="match.name"></p>
          </li>
        </ul>
    </div>
    <span class="highlight"></span>
    <span class="bar"></span>
    <label>{{placeholder}}</label>
  </span>
</template>

<script>
import debounce from 'lodash.debounce'

export default {
  name: 'typeahead',
  props: {
    source: {
      type: Array,
      default: () => { return [] },
      required: true
    },
    onSelect: {
      type: Function,
      required: true
    },
    onChange: {
      type: Function,
      required: true
    },
    limit: {
      type: Number,
      default: 5,
      required: false
    },
    name: {
      type: String,
      required: false
    },
    async: {
      type: Boolean,
      default: true,
      required: false
    },
    error: {
      type: Boolean,
      default: false,
      required: false
    },
    prefill: {
      default: null,
      required: false
    },
    placeholder: {
      type: String,
      default() { return '' },
      required: false
    },
    trigger: {
      default: ()=>{return true},
      required: false
    }
  },
  data () {
    return {
      query: '',
      matches: [],
      hint: '',
      selected: false,
      is_first: false,
      outfocus:{
        input: false,
        suggestion: false
      },
      firstTouch: true,
      preselected: 0
    }
  },
  watch: {
    outfocus:{
      handler: function(n){
        /*if(n.input==true&&n.suggestion==true)
          this.selected=true*/
        //n.input=n.suggestion=false
      }
    },
    prefill: function(){
      if(this.prefill!=null)
        this.query=this.prefill  
    },
    query: function (value) {
      if(value=="")
        this.is_first=true
      if(this.is_first==true && this.trigger(value))  
      { if (this.async) {
          debounce(this.getMatches, 150)(value)
        } else {
          this.getMatches(value)
        }
        this.onChange(value, this.name)
      }
      else
      { this.getMatches("")
        this.$emit('hideSuggestion',true)
        this.onChange(value, this.name)
      }
      this.is_first=true
    }
  },
  methods: {
    handleDelete () {
      this.selected = false
      this.preselected = 0
      this.firstTouch = true
    },
    handleKeyUp () {
      if (this.matches && this.preselected != 0) {
        this.preselected--
        // let el = document.getElementsByClassName("vbta-suggestion")[this.preselected]
        let el = this.$refs.imageContainer[this.preselected]
        el.style['border-color'] = '#ff3860'
        // let prev = document.getElementsByClassName("vbta-suggestion")[this.preselected + 1]
        let prev = this.$refs.imageContainer[this.preselected + 1]
        prev.style['border-color'] = 'transparent'
        /*let el = $('.vbta-suggestion').get(this.preselected)
        $(el).css('background-color', '#00d1b2')
        
        let prev = $('.vbta-suggestion').get(this.preselected + 1)
        $(prev).css('background-color', '#ffffff')*/
      }
    },
    handleKeyDown () {
      if (this.matches && this.preselected != this.matches.length - 1) {
        console.log('rr')
        if (!this.firstTouch) {
          console.log('ss')
          this.preselected++
        } else {
          this.firstTouch = false
        }
        // let el = document.getElementsByClassName("vbta-suggestion")[this.preselected]
        let el = this.$refs.imageContainer[this.preselected]
        el.style['border-color'] = '#ff3860'
        if (this.preselected != 0) {
          // let prev = document.getElementsByClassName("vbta-suggestion")[this.preselected - 1]
          let prev = this.$refs.imageContainer[this.preselected - 1]
          prev.style['border-color'] = 'transparent'
        }
        /*let el = $('.vbta-suggestion').get(this.preselected)
        $(el).css('background-color', '#00d1b2')

        if (this.preselected != 0) {
          let prev = $('.vbta-suggestion').get(this.preselected - 1)
          $(prev).css('background-color', '#ffffff')
        }*/
      }
    },
    emitSelect ({name}) {
      //console.log("emiting")
      name = name.replace(/<[\/]?strong>/gm, '')
      this.selected = true
      this.query = name
      this.onSelect(name, this.name)
    },
    getMatches (query) {
      if (query) {
        let matches = []
        let regex = new RegExp(query, 'i')
        let isMatch = false
        this.source.forEach(({images, name}) => {
          // if (typeof value !== 'string') new TypeError(`Typeahead sources must be string. Received ${typeof value}.`)
          if (matches.length === this.limit) return

          let regexProps = regex.exec(name)
          if (regexProps) {
            isMatch = true

            let substr1 = name.substring(0, regexProps.index)
            let substr2 = `<strong>${name.slice(regexProps.index, regexProps.index + query.length)}</strong>`
            let substr3 = name.substring(regexProps.index + query.length)

            let match = substr1 + substr2 + substr3

            matches.push({
              name : match,
              img_url : images[0].url
            })

            if (regexProps.index == 0) {
              let hint = matches[0].name.replace(/<[\/]?strong>/gm, '').substring(query.length)
              if (hint !== this.hint) this.hint = query + hint
            }
          }
          if (!isMatch) {
            this.hint = ''
          }
        })
        if(matches.length==0||this.selected)
          this.$emit('hideSuggestion',true)
        else
          this.$emit('hideSuggestion',false)
        this.matches = matches
      } else {
        this.hint = ''
        this.matches = []
      }
    }
  }
}
</script>

<style lang="scss" scoped>
@import "~bulma/sass/utilities/_all";
//@import "~bulma/sass/elements/form.sass";
.suggestion-box {
  max-height: 9.55rem;
  overflow-x: scroll;
  //@include overflow-touch;
  -webkit-overflow-scrolling: none;
  ul {
    white-space: nowrap;
  }
}
.suggestion-box::-webkit-scrollbar {
    -webkit-appearance: none;
    display: block;
    height: 6px;
}
.suggestion-box::-webkit-scrollbar-thumb {
    background: $grey-light;
    border-radius: 2.5px;
    height: 6px
}
.vbta {
  width: 100%;
  position: relative;
  display: inline-block;
  .tooltip
  {
    position: absolute;
  }
}

.vbta-input {
  background-color: transparent;
  box-shadow: none;
}

.vbta-menu.visible {
  display: inline-block;
}

.vbta-menu {
  position: absolute;
  width: calc(100vw - 2rem);
  top: 100%;
  right: -3rem;
  z-index: 9;
  display: none;
  float: right;
  min-width: 160px;
  padding: 5px 0;
  margin: 6px 0 0;
  list-style: none;
  font-size: 14px;
  text-align: left;
  background-color: $grey-lighter;
  border: 1px solid #cccccc;
  border: 1px solid rgba(0, 0, 0, 0.15);
  border-radius: 4px;
  -webkit-box-shadow: 0 6px 12px rgba(0, 0, 0, 0.175);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.175);
  background-clip: padding-box;
}
.vbta-suggestion {
  display: inline-block;
  padding: 3px 20px;
  clear: both;
  font-weight: normal;
  line-height: 1.42857143;
  color: #333333;
  /*white-space: nowrap;*/
  .image {
    border-radius : 50%;
    border : 1px solid transparent;
    padding : .2rem;

    img{
      border-radius : 50%;
      background: $white;
    }
  }
}

//.vbta-suggestion:hover,
.vbta-suggestion:active {
  color: #ffffff;
  text-decoration: none;
  outline: 0;
  background-color: #00d1b2;
  cursor: pointer;
}

.vbta-hint {
  color: #999;
  position: absolute;
  display: none !important;
  top: 0px;
  left: 0px;
  border-color: transparent;
  box-shadow: none;
  opacity: 1;
  // background: none 0% 0% / auto repeat scroll padding-box border-box rgb(255, 255, 255);
}
.vbta-suggest:after, .vbta-suggest:before{
    position: absolute;
    display: inline-block;
    border-right: 7px solid transparent;
    border-bottom: 7px solid;
    border-left: 7px solid transparent;
    border-bottom-color:  $grey-lighter;
    content: '';
    z-index: 9;
    right: 50%;
    bottom: -8px;
}
.vbta-suggest:before{
  bottom : -6px;
  border-bottom-color: rgba(0, 0, 0, 0.15);
}
.vbta-hint.visible{
  display: inline-block;
}
</style>
