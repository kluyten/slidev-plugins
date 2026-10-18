<!--
MIT License

Copyright (c) 2025 Kris Luyten

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND...
-->

<template>
    <div class="llm-wrapper">
      <!-- Slide Content - This is what appears on the slide -->
      <div class="slide-content" ref="slideContentRef">
        <slot />
      </div>
      
      <!-- Floating Query Interface -->
      <div class="llm-query-container" :class="positionClass">
        <!-- Trigger Icon -->
        <button 
          @click="toggleQuery" 
          class="llm-icon-btn"
          :class="{ active: showQuery }"
          :title="bufferTitle"
        >
          <component :is="iconComponent" class="llm-icon" />
          <span
            v-if="hasAutoPrompt"
            class="auto-badge"
            :class="bufferState"
            :title="bufferTitle"
          >{{ bufferLabel }}</span>
        </button>
  
        <!-- Query Interface -->
        <div v-if="showQuery" class="llm-interface">
        <div class="llm-header">
          <component :is="iconComponent" class="llm-header-icon" />
          <select
            v-model="selectedModel"
            class="llm-model-select"
            aria-label="Model"
            :title="activeModel"
            @keydown.stop
          >
            <optgroup v-for="g in pickerGroups" :key="g.label" :label="g.label">
              <option v-for="o in g.options" :key="o.id" :value="o.id">{{ o.name }}</option>
            </optgroup>
          </select>
          <div class="header-actions">
            <button v-if="!autoExecuted" @click="executeQuery" class="refresh-btn" title="Ask again">
              ↻
            </button>
            <button @click="showQuery = false" class="close-btn" aria-label="Close">×</button>
          </div>
        </div>
        
        <!-- Query Input (single textarea for both manual and auto prompts) -->
        <div class="query-input">
          <div class="prompt-input-section">
            <textarea
              v-model="currentPrompt"
              :placeholder="hasAutoPrompt ? '' : 'Enter your question...'"
              class="llm-input"
              @keydown.ctrl.enter="executeQuery"
            />

            <button
              @click="executeQuery"
              :disabled="loading || !currentPrompt.trim()"
              class="llm-send-btn"
              :title="loading ? 'Thinking...' : (hasAutoPrompt ? 'Run Query' : 'Ask')"
            >
            </button>
          </div>
        </div>
  
        <!-- Response Display -->
        <div v-if="response || error || loading" ref="responseRef" class="llm-response">
          <div v-if="loading" class="loading">
            <div class="loading-spinner"></div>
            Thinking...
          </div>
          <div v-else-if="error" class="error">
            Error: {{ error }}
          </div>
          <div v-else class="response-content" v-html="formattedResponse">          </div>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { ref, computed, onMounted, nextTick, useSlots } from 'vue'
  import axios from 'axios'
  import { marked } from 'marked'
  
  // Icons for different models retrieved from the UXWing SVG Icons website (https://uxwing.com/)
const ChatGPTIcon = {
  template: `<svg viewBox="0 0 24 24" fill="currentColor"><path d="M22.2819 9.8211a5.9847 5.9847 0 0 0-.5157-4.9108 6.0462 6.0462 0 0 0-6.5098-2.9A6.0651 6.0651 0 0 0 4.9807 4.1818a5.9847 5.9847 0 0 0-3.9977 2.9 6.0462 6.0462 0 0 0 .7427 7.0966 5.98 5.98 0 0 0 .511 4.9107 6.051 6.051 0 0 0 6.5146 2.9001A5.9847 5.9847 0 0 0 13.2599 24a6.0557 6.0557 0 0 0 5.7718-4.2058 5.9894 5.9894 0 0 0 3.9977-2.9001 6.0557 6.0557 0 0 0-.7475-7.0729zm-9.022 12.6081a4.4755 4.4755 0 0 1-2.8764-1.0408l.1419-.0804 4.7783-2.7582a.7948.7948 0 0 0 .3927-.6813v-6.7369l2.02 1.1686a.071.071 0 0 1 .038.052v5.5826a4.504 4.504 0 0 1-4.4945 4.4944zm-9.6607-4.1254a4.4708 4.4708 0 0 1-.5346-3.0137l.142.0852 4.783 2.7582a.7712.7712 0 0 0 .7806 0l5.8428-3.3685v2.3324a.0804.0804 0 0 1-.0332.0615L9.74 19.9502a4.4992 4.4992 0 0 1-6.1408-1.6464zM2.3408 7.8956a4.485 4.485 0 0 1 2.3655-1.9728V11.6a.7664.7664 0 0 0 .3879.6765l5.8144 3.3543-2.0201 1.1685a.0757.0757 0 0 1-.071 0l-4.8303-2.7865A4.504 4.504 0 0 1 2.3408 7.872zm16.5963 3.8558L13.1038 8.364 15.1192 7.2a.0757.0757 0 0 1 .071 0l4.8303 2.7913a4.4944 4.4944 0 0 1-.6765 8.1042v-5.6772a.79.79 0 0 0-.407-.667zm2.0107-3.0231l-.142-.0852-4.7735-2.7818a.7759.7759 0 0 0-.7854 0L9.409 9.2297V6.8974a.0662.0662 0 0 1 .0284-.0615l4.8303-2.7866a4.4992 4.4992 0 0 1 6.6802 4.66zM8.3065 12.863l-2.02-1.1638a.0804.0804 0 0 1-.038-.0567V6.0742a4.4992 4.4992 0 0 1 7.3757-3.4537l-.142.0805L8.704 5.459a.7948.7948 0 0 0-.3927.6813zm1.0976-2.3654l2.602-1.4998 2.6069 1.4998v2.9994l-2.5974 1.4997-2.6067-1.4997Z"/></svg>`
}

const ClaudeIcon = {
  template: `<svg xmlns="http://www.w3.org/2000/svg" shape-rendering="geometricPrecision" text-rendering="geometricPrecision" image-rendering="optimizeQuality" fill-rule="evenodd" clip-rule="evenodd" viewBox="0 0 512 509.64"><path fill="#D77655" d="M115.612 0h280.775C459.974 0 512 52.026 512 115.612v278.415c0 63.587-52.026 115.612-115.613 115.612H115.612C52.026 509.639 0 457.614 0 394.027V115.612C0 52.026 52.026 0 115.612 0z"/><path fill="#FCF2EE" fill-rule="nonzero" d="M142.27 316.619l73.655-41.326 1.238-3.589-1.238-1.996-3.589-.001-12.31-.759-42.084-1.138-36.498-1.516-35.361-1.896-8.897-1.895-8.34-10.995.859-5.484 7.482-5.03 10.717.935 23.683 1.617 35.537 2.452 25.782 1.517 38.193 3.968h6.064l.86-2.451-2.073-1.517-1.618-1.517-36.776-24.922-39.81-26.338-20.852-15.166-11.273-7.683-5.687-7.204-2.451-15.721 10.237-11.273 13.75.935 3.513.936 13.928 10.716 29.749 23.027 38.848 28.612 5.687 4.727 2.275-1.617.278-1.138-2.553-4.271-21.13-38.193-22.546-38.848-10.035-16.101-2.654-9.655c-.935-3.968-1.617-7.304-1.617-11.374l11.652-15.823 6.445-2.073 15.545 2.073 6.547 5.687 9.655 22.092 15.646 34.78 24.265 47.291 7.103 14.028 3.791 12.992 1.416 3.968 2.449-.001v-2.275l1.997-26.641 3.69-32.707 3.589-42.084 1.239-11.854 5.863-14.206 11.652-7.683 9.099 4.348 7.482 10.716-1.036 6.926-4.449 28.915-8.72 45.294-5.687 30.331h3.313l3.792-3.791 15.342-20.372 25.782-32.227 11.374-12.789 13.27-14.129 8.517-6.724 16.1-.001 11.854 17.617-5.307 18.199-16.581 21.029-13.75 17.819-19.716 26.54-12.309 21.231 1.138 1.694 2.932-.278 44.536-9.479 24.062-4.347 28.714-4.928 12.992 6.066 1.416 6.167-5.106 12.613-30.71 7.583-36.018 7.204-53.636 12.689-.657.48.758.935 24.164 2.275 10.337.556h25.301l47.114 3.514 12.309 8.139 7.381 9.959-1.238 7.583-18.957 9.655-25.579-6.066-59.702-14.205-20.474-5.106-2.83-.001v1.694l17.061 16.682 31.266 28.233 39.152 36.397 1.997 8.999-5.03 7.102-5.307-.758-34.401-25.883-13.27-11.651-30.053-25.302-1.996-.001v2.654l6.926 10.136 36.574 54.975 1.895 16.859-2.653 5.485-9.479 3.311-10.414-1.895-21.408-30.054-22.092-33.844-17.819-30.331-2.173 1.238-10.515 113.261-4.929 5.788-11.374 4.348-9.478-7.204-5.03-11.652 5.03-23.027 6.066-30.052 4.928-23.886 4.449-29.674 2.654-9.858-.177-.657-2.173.278-22.37 30.71-34.021 45.977-26.919 28.815-6.445 2.553-11.173-5.789 1.037-10.337 6.243-9.2 37.257-47.392 22.47-29.371 14.508-16.961-.101-2.451h-.859l-98.954 64.251-17.618 2.275-7.583-7.103.936-11.652 3.589-3.791 29.749-20.474-.101.102.024.101z"/></svg>
    `
}
  

const GeminiIcon = {
    template:
`<svg fill="none" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 65 65"><mask id="maskme" style="mask-type:alpha" maskUnits="userSpaceOnUse" x="0" y="0" width="65" height="65"><path d="M32.447 0c.68 0 1.273.465 1.439 1.125a38.904 38.904 0 001.999 5.905c2.152 5 5.105 9.376 8.854 13.125 3.751 3.75 8.126 6.703 13.125 8.855a38.98 38.98 0 005.906 1.999c.66.166 1.124.758 1.124 1.438 0 .68-.464 1.273-1.125 1.439a38.902 38.902 0 00-5.905 1.999c-5 2.152-9.375 5.105-13.125 8.854-3.749 3.751-6.702 8.126-8.854 13.125a38.973 38.973 0 00-2 5.906 1.485 1.485 0 01-1.438 1.124c-.68 0-1.272-.464-1.438-1.125a38.913 38.913 0 00-2-5.905c-2.151-5-5.103-9.375-8.854-13.125-3.75-3.749-8.125-6.702-13.125-8.854a38.973 38.973 0 00-5.905-2A1.485 1.485 0 010 32.448c0-.68.465-1.272 1.125-1.438a38.903 38.903 0 005.905-2c5-2.151 9.376-5.104 13.125-8.854 3.75-3.749 6.703-8.125 8.855-13.125a38.972 38.972 0 001.999-5.905A1.485 1.485 0 0132.447 0z" fill="#000"/><path d="M32.447 0c.68 0 1.273.465 1.439 1.125a38.904 38.904 0 001.999 5.905c2.152 5 5.105 9.376 8.854 13.125 3.751 3.75 8.126 6.703 13.125 8.855a38.98 38.98 0 005.906 1.999c.66.166 1.124.758 1.124 1.438 0 .68-.464 1.273-1.125 1.439a38.902 38.902 0 00-5.905 1.999c-5 2.152-9.375 5.105-13.125 8.854-3.749 3.751-6.702 8.126-8.854 13.125a38.973 38.973 0 00-2 5.906 1.485 1.485 0 01-1.438 1.124c-.68 0-1.272-.464-1.438-1.125a38.913 38.913 0 00-2-5.905c-2.151-5-5.103-9.375-8.854-13.125-3.75-3.749-8.125-6.702-13.125-8.854a38.973 38.973 0 00-5.905-2A1.485 1.485 0 010 32.448c0-.68.465-1.272 1.125-1.438a38.903 38.903 0 005.905-2c5-2.151 9.376-5.104 13.125-8.854 3.75-3.749 6.703-8.125 8.855-13.125a38.972 38.972 0 001.999-5.905A1.485 1.485 0 0132.447 0z" fill="url(#prefix__paint0_linear_2001_67)"/></mask><g mask="url(#maskme)"><g filter="url(#prefix__filter0_f_2001_67)"><path d="M-5.859 50.734c7.498 2.663 16.116-2.33 19.249-11.152 3.133-8.821-.406-18.131-7.904-20.794-7.498-2.663-16.116 2.33-19.25 11.151-3.132 8.822.407 18.132 7.905 20.795z" fill="#FFE432"/></g><g filter="url(#prefix__filter1_f_2001_67)"><path d="M27.433 21.649c10.3 0 18.651-8.535 18.651-19.062 0-10.528-8.35-19.062-18.651-19.062S8.78-7.94 8.78 2.587c0 10.527 8.35 19.062 18.652 19.062z" fill="#FC413D"/></g><g filter="url(#prefix__filter2_f_2001_67)"><path d="M20.184 82.608c10.753-.525 18.918-12.244 18.237-26.174-.68-13.93-9.95-24.797-20.703-24.271C6.965 32.689-1.2 44.407-.519 58.337c.681 13.93 9.95 24.797 20.703 24.271z" fill="#00B95C"/></g><g filter="url(#prefix__filter3_f_2001_67)"><path d="M20.184 82.608c10.753-.525 18.918-12.244 18.237-26.174-.68-13.93-9.95-24.797-20.703-24.271C6.965 32.689-1.2 44.407-.519 58.337c.681 13.93 9.95 24.797 20.703 24.271z" fill="#00B95C"/></g><g filter="url(#prefix__filter4_f_2001_67)"><path d="M30.954 74.181c9.014-5.485 11.427-17.976 5.389-27.9-6.038-9.925-18.241-13.524-27.256-8.04-9.015 5.486-11.428 17.977-5.39 27.902 6.04 9.924 18.242 13.523 27.257 8.038z" fill="#00B95C"/></g><g filter="url(#prefix__filter5_f_2001_67)"><path d="M67.391 42.993c10.132 0 18.346-7.91 18.346-17.666 0-9.757-8.214-17.667-18.346-17.667s-18.346 7.91-18.346 17.667c0 9.757 8.214 17.666 18.346 17.666z" fill="#3186FF"/></g><g filter="url(#prefix__filter6_f_2001_67)"><path d="M-13.065 40.944c9.33 7.094 22.959 4.869 30.442-4.972 7.483-9.84 5.987-23.569-3.343-30.663C4.704-1.786-8.924.439-16.408 10.28c-7.483 9.84-5.986 23.57 3.343 30.664z" fill="#FBBC04"/></g><g filter="url(#prefix__filter7_f_2001_67)"><path d="M34.74 51.43c11.135 7.656 25.896 5.524 32.968-4.764 7.073-10.287 3.779-24.832-7.357-32.488C49.215 6.52 34.455 8.654 27.382 18.94c-7.072 10.288-3.779 24.833 7.357 32.49z" fill="#3186FF"/></g><g filter="url(#prefix__filter8_f_2001_67)"><path d="M54.984-2.336c2.833 3.852-.808 11.34-8.131 16.727-7.324 5.387-15.557 6.631-18.39 2.78-2.833-3.853.807-11.342 8.13-16.728 7.324-5.387 15.558-6.631 18.39-2.80z" fill="#749BFF"/></g><g filter="url(#prefix__filter9_f_2001_67)"><path d="M31.727 16.104C43.053 5.598 46.94-8.626 40.41-15.666c-6.53-7.04-21.006-4.232-32.332 6.274s-15.214 24.73-8.683 31.77c6.53 7.04 21.006 4.232 32.332-6.274z" fill="#FC413D"/></g><g filter="url(#prefix__filter10_f_2001_67)"><path d="M8.51 53.838c6.732 4.818 14.46 5.55 17.262 1.636 2.802-3.915-.384-10.994-7.116-15.812-6.731-4.818-14.46-5.55-17.261-1.636-2.802 3.915.383 10.994 7.115 15.812z" fill="#FFEE48"/></g></g><defs><filter id="prefix__filter0_f_2001_67" x="-19.824" y="13.152" width="39.274" height="43.217" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="2.46" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter1_f_2001_67" x="-15.001" y="-40.257" width="84.868" height="85.688" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="11.891" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter2_f_2001_67" x="-20.776" y="11.927" width="79.454" height="90.916" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="10.109" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter3_f_2001_67" x="-20.776" y="11.927" width="79.454" height="90.916" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="10.109" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter4_f_2001_67" x="-19.845" y="15.459" width="79.731" height="81.505" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="10.109" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter5_f_2001_67" x="29.832" y="-11.552" width="75.117" height="73.758" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="9.606" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter6_f_2001_67" x="-38.583" y="-16.253" width="78.135" height="78.758" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="8.706" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter7_f_2001_67" x="8.107" y="-5.966" width="78.877" height="77.539" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="7.775" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter8_f_2001_67" x="-13.587" y="-18.488" width="56.272" height="51.81" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="6.957" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter9_f_2001_67" x="-15.526" y="-31.297" width="70.856" height="69.306" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="5.876" result="effect1_foregroundBlur_2001_67"/></filter><filter id="prefix__filter10_f_2001_67" x="-14.168" y="20.964" width="55.501" height="51.571" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB"><feFlood flood-opacity="0" result="BackgroundImageFix"/><feBlend in="SourceGraphic" in2="BackgroundImageFix" result="shape"/><feGaussianBlur stdDeviation="7.273" result="effect1_foregroundBlur_2001_67"/></filter><linearGradient id="prefix__paint0_linear_2001_67" x1="18.447" y1="43.42" x2="52.153" y2="15.004" gradientUnits="userSpaceOnUse"><stop stop-color="#4893FC"/><stop offset=".27" stop-color="#4893FC"/><stop offset=".777" stop-color="#969DFF"/><stop offset="1" stop-color="#BD99FE"/></linearGradient></defs></svg>`
}


const LlamaIcon = {
  template: `<svg viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg" fill-rule="evenodd" clip-rule="evenodd" stroke-linejoin="round" stroke-miterlimit="2"><path d="M12 0c6.627 0 12 5.373 12 12s-5.373 12-12 12S0 18.627 0 12 5.373 0 12 0zm0 3.627c-4.593 0-8.373 3.78-8.373 8.373s3.78 8.373 8.373 8.373 8.373-3.78 8.373-8.373S16.593 3.627 12 3.627z" fill="url(#prefix___Linear1)" transform="scale(21.33334)"/><defs><linearGradient id="prefix___Linear1" x1="0" y1="0" x2="1" y2="0" gradientUnits="userSpaceOnUse" gradientTransform="matrix(-24 24 -24 -24 24 0)"><stop offset="0" stop-color="#ff97e3"/><stop offset=".13" stop-color="#ff97e3"/><stop offset=".18" stop-color="#d14fe1"/><stop offset=".34" stop-color="#0050e2"/><stop offset=".67" stop-color="#0050e2"/><stop offset=".81" stop-color="#00ddf4"/><stop offset=".86" stop-color="#23f8cc"/><stop offset="1" stop-color="#23f8cc"/></linearGradient></defs></svg>`
}

const GrokIcon = {
  template: `<svg fill="none" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 492"><path fill-rule="evenodd" clip-rule="evenodd" d="M197.76 315.52l170.197-125.803c8.342-6.186 20.267-3.776 24.256 5.803 20.907 50.539 11.563 111.253-30.08 152.939-41.621 41.685-99.562 50.816-152.512 29.994l-57.834 26.816c82.965 56.768 183.701 42.731 246.656-20.33 49.941-50.006 65.408-118.166 50.944-179.627l.128.149c-20.971-90.282 5.162-126.378 58.666-200.17 1.28-1.75 2.56-3.499 3.819-5.291l-70.421 70.507v-.214l-243.883 245.27m-35.072 30.528c-59.563-56.96-49.28-145.088 1.515-195.926 37.568-37.61 99.136-52.97 152.874-30.4l57.707-26.666a166.554 166.554 0 00-39.019-21.334 191.467 191.467 0 00-208.042 41.942c-54.038 54.101-71.04 137.301-41.856 208.298 21.802 53.056-13.931 90.582-49.92 128.47C23.104 463.915 10.304 477.333 0 491.541l162.56-145.386" fill="#000"/></svg>`
}


const MistralIcon = { template: `<svg viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg" fill-rule="evenodd" clip-rule="evenodd" stroke-linejoin="round" stroke-miterlimit="2"><path d="M189.08 303.228H94.587l.044-94.446h94.497l-.048 94.446z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M283.528 397.674h-94.493l.044-94.446h94.496l-.047 94.446z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M283.575 303.228H189.08l.046-94.446h94.496l-.047 94.446z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M378.07 303.228h-94.495l.044-94.446h94.498l-.047 94.446zM189.128 208.779H94.633l.044-94.448h94.498l-.047 94.448zM378.115 208.779h-94.494l.045-94.448h94.496l-.047 94.448zM94.587 303.227H.093l.044-96.017h94.496l-.046 96.017z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M94.633 208.779H.138l.046-94.448H94.68l-.047 94.448z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M94.68 115.902H.185L.23 19.885h94.498l-.047 96.017zM472.657 114.331h-94.495l.044-94.446h94.497l-.046 94.446zM94.54 399.244H.046l.044-97.588h94.497l-.047 97.588z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M94.495 492.123H0l.044-94.446H94.54l-.045 94.446zM472.563 303.228H378.07l.044-94.446h94.496l-.047 94.446zM472.61 208.779h-94.495l.044-94.448h94.498l-.047 94.448z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M472.517 397.674h-94.494l.044-94.446h94.497l-.047 94.446z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M472.47 492.121h-94.493l.044-96.017h94.496l-.047 96.017z" fill="#1c1c1b" fill-rule="nonzero"/><path d="M228.375 303.22h-96.061l.046-94.446h96.067l-.052 94.446z" fill="#ff7000" fill-rule="nonzero"/><path d="M322.827 397.666h-94.495l.044-96.018h94.498l-.047 96.018z" fill="#ff4900" fill-rule="nonzero"/><path d="M324.444 303.22h-97.636l.046-94.446h97.638l-.048 94.446z" fill="#ff7000" fill-rule="nonzero"/><path d="M418.938 303.22h-96.064l.045-94.446h96.066l-.047 94.446z" fill="#ff7000" fill-rule="nonzero"/><path d="M228.423 208.77H132.36l.045-94.445h96.066l-.05 94.446zM418.985 208.77H322.92l.044-94.445h96.069l-.048 94.446z" fill="#ffa300" fill-rule="nonzero"/><path d="M133.883 304.79H39.392l.044-96.017h94.496l-.049 96.017z" fill="#ff7000" fill-rule="nonzero"/><path d="M133.929 208.77H39.437l.044-95.445h94.496l-.048 95.445z" fill="#ffa300" fill-rule="nonzero"/><path d="M133.976 114.325H39.484l.044-94.448h94.497l-.05 94.448zM511.954 115.325h-94.493l.044-95.448h94.497l-.048 95.448z" fill="#ffce00" fill-rule="nonzero"/><path d="M133.836 399.667H39.345l.044-96.447h94.496l-.049 96.447z" fill="#ff4900" fill-rule="nonzero"/><path d="M133.79 492.117H39.3l.044-94.448h94.496l-.049 94.448z" fill="#ff0107" fill-rule="nonzero"/><path d="M511.862 303.22h-94.495l.046-94.446h94.496l-.047 94.446z" fill="#ff7000" fill-rule="nonzero"/><path d="M511.907 208.77h-94.493l.044-94.445h94.496l-.047 94.446z" fill="#ffa300" fill-rule="nonzero"/><path d="M511.815 398.666h-94.493l.044-95.447h94.496l-.047 95.447z" fill="#ff4900" fill-rule="nonzero"/><path d="M511.77 492.117h-94.496l.046-94.448h94.496l-.047 94.448z" fill="#ff0107" fill-rule="nonzero"/></svg>`
}
  
const DeepSeekIcon = {
  template: `<svg viewBox="0 0 24 24" fill="currentColor"><path d="M12 2a10 10 0 1 0 0 20 10 10 0 0 0 0-20zm0 3.2a6.8 6.8 0 0 1 6.55 5h-4.2A3.2 3.2 0 0 0 12 8.2a3.2 3.2 0 0 0-2.35 1H5.45A6.8 6.8 0 0 1 12 5.2zm-6.8 6.8c0-.27.02-.54.05-.8h4.3a3.2 3.2 0 0 0 0 1.6h-4.3a6.9 6.9 0 0 1-.05-.8zm6.8 6.8a6.8 6.8 0 0 1-6.15-3.85h4.2A3.2 3.2 0 0 0 12 15.8a3.2 3.2 0 0 0 1.95-.65h4.2A6.8 6.8 0 0 1 12 18.8z"/></svg>`
}

const MoonshotIcon = {
  template: `<svg viewBox="0 0 24 24" fill="currentColor"><path d="M17.3 14.8A7.2 7.2 0 0 1 8 5.6a.8.8 0 0 0-1.2-.85A8.8 8.8 0 1 0 19 17.9a.8.8 0 0 0-.85-1.2 7.3 7.3 0 0 1-.85.1z"/><circle cx="17.5" cy="6.5" r="2.2"/></svg>`
}

const QwenIcon = {
  template: `<svg viewBox="0 0 24 24" fill="currentColor"><path d="M12 2 3 7v10l9 5 9-5V7l-9-5zm0 2.3 6.8 3.8-6.8 3.8L5.2 8.1 12 4.3zM5 9.9l6 3.4v6.8l-6-3.3V9.9zm14 0v6.9l-6 3.3v-6.8l6-3.4z"/></svg>`
}

// Shown when the model id is not in modelConfigs - makes a typo or a retired
// model obvious at a glance instead of silently falling back to another logo.
const UnknownIcon = {
  template: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="10"/><path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"/><line x1="12" y1="17" x2="12.01" y2="17"/></svg>`
}

  const props = withDefaults(defineProps<{
    // Optional. When a slide names no model, the panel shows the picker and
    // starts on DEFAULT_MODEL, so a deck can be written without choosing one.
    model?: string
    apiKey?: string
    systemPrompt?: string
    prompt?: string
    position?: 'top-right' | 'top-left' | 'bottom-right' | 'bottom-left'
    // Vue casts an ABSENT Boolean prop to false, not undefined, so a bare
    // `props.autoExecute !== false` test never holds and the buffered fetch
    // never runs. The default has to be declared explicitly.
    autoExecute?: boolean
  }>(), {
    autoExecute: true,
  })

  const slots = useSlots()

  const showQuery = ref(false)
  const currentPrompt = ref('')
  const response = ref('')
  const loading = ref(false)
  const error = ref('')
  const slotContent = ref('')
  const autoExecuted = ref(false)
  const slideContentRef = ref<HTMLElement | null>(null)
  
  // Model roster, verified against the OpenRouter catalogue.
  // The six "fast" entries are the low-latency tier of each provider and are what
  // the slides use by default; 'deep' models are reserved for critique prompts,
  // where the answer is worth a few seconds more.
  // Prices are USD per million tokens (input / output) at the time of writing.
  const modelConfigs = {
    // --- fast tier: use these for "generate an example" prompts ---
    'google/gemini-3.1-flash-lite':  { name: 'Gemini 3.1 Flash Lite', icon: GeminiIcon,  tier: 'fast' }, // 0.25 / 1.50
    'openai/gpt-5.6-luna':           { name: 'GPT-5.6 Luna',          icon: ChatGPTIcon, tier: 'fast' }, // 0.20 / 1.20
    'meta-llama/llama-4-scout':      { name: 'Llama 4 Scout',         icon: LlamaIcon,   tier: 'fast' }, // 0.10 / 0.30
    'mistralai/ministral-8b-2512':   { name: 'Ministral 8B',          icon: MistralIcon, tier: 'fast' }, // 0.15 / 0.15
    'anthropic/claude-haiku-4.5':    { name: 'Claude Haiku 4.5',      icon: ClaudeIcon,  tier: 'fast' }, // 1.00 / 5.00

    // --- deep tier: use sparingly, for "criticise this" / "argue with the professor" ---
    'anthropic/claude-sonnet-5':     { name: 'Claude Sonnet 5',       icon: ClaudeIcon,  tier: 'general' }, // 2.00 / 10.00
    'google/gemini-3.6-flash':       { name: 'Gemini 3.6 Flash',      icon: GeminiIcon,  tier: 'general' }, // 0.75 / 3.75
    'openai/gpt-4.1':                { name: 'GPT-4.1',               icon: ChatGPTIcon, tier: 'general' }, // 2.00 / 8.00

    // --- strong reasoning: slower and dearer, for "criticise this" / "argue with me" ---
    'anthropic/claude-opus-5':       { name: 'Claude Opus 5',         icon: ClaudeIcon,   tier: 'reasoning' }, //  5.00 / 25.00
    'openai/gpt-5.6-terra-pro':      { name: 'GPT-5.6 Terra Pro',     icon: ChatGPTIcon,  tier: 'reasoning' }, //  2.00 / 12.00
    'x-ai/grok-4.6':                 { name: 'Grok 4.6',              icon: GrokIcon,     tier: 'reasoning' }, //  2.00 /  6.00
    'moonshotai/kimi-k3':            { name: 'Kimi K3',               icon: MoonshotIcon, tier: 'reasoning' }, //  1.70 /  8.50

    // --- general purpose ---
    'x-ai/grok-4.3':                 { name: 'Grok 4.3',              icon: GrokIcon,     tier: 'general' },   //  1.25 /  2.50
    'deepseek/deepseek-v4-pro-0813': { name: 'DeepSeek V4 Pro',       icon: DeepSeekIcon, tier: 'general' },   //  0.66 /  1.98

    // --- coding ---
    'openai/gpt-5.3-codex':          { name: 'GPT-5.3 Codex',         icon: ChatGPTIcon,  tier: 'coding' },    //  1.75 / 14.00
    'qwen/qwen3-coder-flash':        { name: 'Qwen3 Coder Flash',     icon: QwenIcon,     tier: 'coding' },    //  0.20 /  0.97
    'mistralai/devstral-2512':       { name: 'Devstral 2512',         icon: MistralIcon,  tier: 'coding' }     //  0.40 /  2.00
  }

  // The dropdown, grouped. Every id verified against openrouter.ai/api/v1/models.
  // Providers retire ids without warning - a stale id returns a 404 in front of the
  // room - so re-check before each term with:
  //   curl -s https://openrouter.ai/api/v1/models | \
  //     python3 -c "import json,sys;print('\n'.join(sorted(m['id'] for m in json.load(sys.stdin)['data'])))"
  const PICKER_GROUPS = [
    { label: 'Reasoning - slower, for argument and critique', ids: [
      'anthropic/claude-opus-5', 'openai/gpt-5.6-terra-pro', 'x-ai/grok-4.6', 'moonshotai/kimi-k3' ] },
    { label: 'General purpose', ids: [
      'anthropic/claude-sonnet-5', 'google/gemini-3.6-flash', 'x-ai/grok-4.3', 'deepseek/deepseek-v4-pro-0813' ] },
    { label: 'Fast - for live examples', ids: [
      'anthropic/claude-haiku-4.5', 'openai/gpt-5.6-luna', 'google/gemini-3.1-flash-lite', 'meta-llama/llama-4-scout' ] },
    { label: 'Coding', ids: [
      'openai/gpt-5.3-codex', 'qwen/qwen3-coder-flash', 'mistralai/devstral-2512' ] },
  ]

  // Models retired by OpenRouter, mapped to their nearest replacement. props.model is
  // resolved through this, so a deck that still names a dead id keeps working instead
  // of returning a 404 mid-lecture.
  const RETIRED = {
    'x-ai/grok-code-fast-1': 'qwen/qwen3-coder-flash',
    'x-ai/grok-4': 'x-ai/grok-4.3',
    'anthropic/claude-3.5-sonnet': 'anthropic/claude-sonnet-5',
    'anthropic/claude-sonnet-4': 'anthropic/claude-sonnet-5',
    'google/gemini-2.5-pro': 'google/gemini-3.6-flash',
    'gpt-4': 'openai/gpt-4.1',
    'mistralai/mistral-small-3.2-24b-instruct:free': 'mistralai/devstral-2512',
  }
  const resolveModel = (id) => RETIRED[id] || id
  
  const DEFAULT_MODEL = 'anthropic/claude-haiku-4.5'


  // The id this panel will actually send. A slide's `model` wins until somebody
  // picks another one in the dropdown; the pick is per panel and per page load,
  // so the four-model comparison slides keep four different answers and a reload
  // returns every slide to what it ships with.
  const chosenModel = ref<string | null>(null)
  const activeModel = computed(() =>
    chosenModel.value || resolveModel(props.model || '') || DEFAULT_MODEL)

  // v-model target for the dropdown: reads the active id, writes the per-panel pick.
  const selectedModel = computed({
    get: () => activeModel.value,
    set: (v: string) => { chosenModel.value = v },
  })

  const isKnownModel = computed(() => Boolean(modelConfigs[activeModel.value]))
  const modelName = computed(() => modelConfigs[activeModel.value]?.name || activeModel.value)
  const iconComponent = computed(() => modelConfigs[activeModel.value]?.icon || UnknownIcon)

  // The dropdown. An id that is selected but absent from the roster is added under
  // its own id, so a slide naming something unusual still shows what it will send.
  const pickerGroups = computed(() => {
    const groups = PICKER_GROUPS.map(g => ({
      label: g.label,
      options: g.ids.map(id => ({ id, name: modelConfigs[id]?.name || id })),
    }))
    const listed = PICKER_GROUPS.some(g => g.ids.includes(activeModel.value))
    if (!listed) {
      groups.unshift({
        label: 'This slide',
        options: [{ id: activeModel.value, name: modelName.value }],
      })
    }
    return groups
  })
  const hasAutoPrompt = computed(() => Boolean(props.prompt))

  // The badge tells you, without opening anything, whether the buffered answer
  // has arrived - so you know if it is safe to click mid-sentence.
  const bufferState = computed(() => {
    if (loading.value) return 'is-loading'
    if (error.value) return 'is-error'
    if (response.value) return 'is-ready'
    return 'is-idle'
  })
  const bufferLabel = computed(() => ({
    'is-loading': '...', 'is-error': '!', 'is-ready': '\u2713', 'is-idle': 'Auto'
  }[bufferState.value]))
  const bufferTitle = computed(() => ({
    'is-loading': `${modelName.value} is thinking...`,
    'is-error': `${modelName.value} failed - click to see why`,
    'is-ready': `${modelName.value} has an answer ready - click to show`,
    'is-idle': `Ask ${modelName.value}`
  }[bufferState.value]))
  
  const positionClass = computed(() => {
    switch (props.position) {
      case 'top-left': return 'pos-top-left'
      case 'bottom-right': return 'pos-bottom-right'
      case 'bottom-left': return 'pos-bottom-left'
      default: return 'pos-top-right'
    }
  })
  
  const formattedResponse = computed(() => {
    if (!response.value) return ''
    // Configure marked for proper markdown rendering
    marked.setOptions({
      breaks: true,
      gfm: true,
      headerIds: false,
      mangle: false
    })
    // Parse markdown to HTML
    return marked.parse(response.value) as string
  })
  
  
  const extractSlotContent = () => {
    // Get the slot content from the rendered DOM to ensure we capture all content types
    if (slideContentRef.value) {
      // Extract all text content from the slide content, preserving structure
      const content = slideContentRef.value.textContent || slideContentRef.value.innerText || ''
      return content.trim()
    }
    return ''
  }
  
  const toggleQuery = async () => {
    showQuery.value = !showQuery.value
    if (showQuery.value) {
      // The buffered response was rendered while the panel was closed, so its
      // scroll height could not be measured until now.
      await nextTick()
      checkScrollable()
    } else if (!hasAutoPrompt.value) {
      response.value = ''
      error.value = ''
    }
  }
  
  const buildFullPrompt = (userPrompt?: string) => {
    const basePrompt = userPrompt || props.prompt || ''

    // Always include slot content if it exists, regardless of whether there's a base prompt
    if (slotContent.value) {
      if (basePrompt) {
        return `${basePrompt}\n\n--- Content from tags ---\n\n${slotContent.value}`
      } else {
        return slotContent.value
      }
    }

    return basePrompt
  }
  
  // Slidev's PDF/PNG export loads every slide in a headless browser. Without this
  // guard each LLMQuery would fire a live (paid) request during every export, and
  // the answer would not even appear - base.css hides the panel when printing.
  const isExporting = () => {
    try {
      const search = window.location.search || ''
      return search.includes('print') || search.includes('export')
    } catch {
      return false
    }
  }

  const responseRef = ref<HTMLElement | null>(null)

  const checkScrollable = () => {
    // Check if response container is scrollable
    if (responseRef.value) {
      const hasScroll = responseRef.value.scrollHeight > responseRef.value.clientHeight
      if (hasScroll) {
        responseRef.value.classList.add('has-scroll')
      } else {
        responseRef.value.classList.remove('has-scroll')
      }
    }
  }

  const executeQuery = async () => {
    if (!currentPrompt.value?.trim()) return

    loading.value = true
    error.value = ''
    response.value = ''

    try {
      const apiKey = props.apiKey || (import.meta.env as any).VITE_OPENROUTER_API_KEY

      if (!apiKey) {
        throw new Error('OpenRouter API key not provided (set VITE_OPENROUTER_API_KEY in .env)')
      }

      if (!isKnownModel.value) {
        console.warn(`[LLMQuery] "${activeModel.value}" is not in modelConfigs - it may have been retired by OpenRouter.`)
      }

      const messages = []

      if (props.systemPrompt) {
        messages.push({ role: 'system', content: props.systemPrompt })
      }

      // Always build the full prompt with slot content for sending to LLM
      const fullPrompt = buildFullPrompt(currentPrompt.value)
      messages.push({ role: 'user', content: fullPrompt })

      const result = await axios.post('https://openrouter.ai/api/v1/chat/completions', {
        model: activeModel.value,
        messages,
        max_tokens: 1500,
        temperature: 0.7
      }, {
        headers: {
          'Authorization': `Bearer ${apiKey}`,
          'Content-Type': 'application/json',
          'HTTP-Referer': window.location.origin,
          'X-Title': 'Sli.dev Presentation'
        }
      })

      response.value = result.data.choices[0]?.message?.content || 'No response received'

      // Mark as auto-executed if this was triggered by autoprompt
      if (hasAutoPrompt.value && currentPrompt.value === props.prompt) {
        autoExecuted.value = true
      }

      // Check for scrollable content after response is set
      await nextTick()
      checkScrollable()
    } catch (err: any) {
      error.value = err.response?.data?.error?.message || err.message || 'Failed to get response'
    } finally {
      loading.value = false
    }
  }
  
  onMounted(async () => {
    // Wait for the DOM to render
    await nextTick()

    // Extract slot content from rendered DOM
    slotContent.value = extractSlotContent()

    // If there's an autoprompt, set it in the textarea and auto-execute
    if (props.prompt) {
      currentPrompt.value = props.prompt

      // Pre-fetch the answer so it is ready the moment you click, but never while
      // exporting. The panel deliberately stays CLOSED: auto-execute is buffering,
      // not a popup. Opening it here would cover the slide you are teaching from.
      if (props.autoExecute !== false && !isExporting()) {
        executeQuery()
      }
    }

    // Check for scrollable content initially
    checkScrollable()
  })
  </script>
  
  <style scoped>
  .llm-wrapper {
    position: relative;
    display: contents; /* This makes the wrapper transparent for layout */
  }
  
  .slide-content {
    display: contents;
  }
  
  .llm-query-container {
    position: fixed;
    z-index: 100;
  }
  
  .pos-top-right {
    top: 20px;
    right: 20px;
  }
  
  .pos-top-left {
    top: 20px;
    left: 20px;
  }
  
  .pos-bottom-right {
    bottom: 20px;
    right: 20px;
  }
  
  .pos-bottom-left {
    bottom: 20px;
    left: 20px;
  }
  
  .llm-icon-btn {
    width: 50px;
    height: 50px;
    border-radius: 50%;
    border: none;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    cursor: pointer;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
  }
  
  .llm-icon-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(0, 0, 0, 0.25);
  }
  
  .llm-icon-btn.active {
    background: linear-gradient(135deg, #764ba2 0%, #667eea 100%);
  }
  
  .llm-icon {
    width: 24px;
    height: 24px;
  }
  
  .auto-badge {
    position: absolute;
    top: -5px;
    right: -5px;
    background: #6b7280;
    color: white;
    border-radius: 8px;
    font-size: 8px;
    padding: 2px 4px;
    font-weight: bold;
    line-height: 1;
    transition: background-color 0.2s ease;
  }

  .auto-badge.is-loading {
    background: #d97706;
    animation: auto-badge-pulse 1.2s ease-in-out infinite;
  }

  .auto-badge.is-ready {
    background: #28a745;
  }

  .auto-badge.is-error {
    background: #dc2626;
  }

  @keyframes auto-badge-pulse {
    0%, 100% { opacity: 1; }
    50%      { opacity: 0.45; }
  }
  
  /* Sized in slide pixels against Slidev's 980x552 canvas. Viewport units are
     wrong here: the slide is a scaled, transformed box, so vh/vw drift with the
     window and a panel that fits on a laptop overflows on a projector. */
  .llm-interface {
    position: absolute;
    top: 58px;
    right: 0;
    width: 660px;
    max-width: calc(100vw - 40px);
    display: flex;
    flex-direction: column;
    max-height: 400px;
    background: #ffffff;
    border-radius: 14px;
    box-shadow: 0 18px 48px rgba(15, 26, 34, 0.22), 0 2px 6px rgba(15, 26, 34, 0.10);
    border: 1px solid #dfe4ea;
    overflow: hidden;
  }

  :global(html.dark) .llm-interface {
    background: #1b232b;
    border-color: #35414d;
    box-shadow: 0 18px 48px rgba(0, 0, 0, 0.55);
  }
  
  .pos-top-left .llm-interface,
  .pos-bottom-left .llm-interface {
    right: auto;
    left: 0;
  }
  
  .pos-bottom-right .llm-interface,
  .pos-bottom-left .llm-interface {
    top: auto;
    bottom: 60px;
  }
  
  .llm-header {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 9px 11px;
    background: #f5f7f9;
    border-bottom: 1px solid #e3e8ed;
    flex: 0 0 auto;
  }

  :global(html.dark) .llm-header {
    background: #212b34;
    border-bottom-color: #35414d;
  }

  .llm-header-icon {
    width: 17px;
    height: 17px;
    flex: 0 0 auto;
    color: #55606b;
  }

  :global(html.dark) .llm-header-icon { color: #9fb0bd; }

  /* The picker carries the header, so the model in use is always readable and
     always changeable - including on a slide that named one. */
  .llm-model-select {
    flex: 1 1 auto;
    min-width: 0;
    font: inherit;
    font-size: 12.5px;
    font-weight: 600;
    color: #1f2933;
    background: transparent;
    border: 1px solid transparent;
    border-radius: 7px;
    padding: 3px 6px;
    cursor: pointer;
  }

  .llm-model-select:hover { background: #e9eef3; border-color: #d3dbe3; }
  .llm-model-select:focus-visible { outline: 2px solid #3b82f6; outline-offset: 1px; }

  :global(html.dark) .llm-model-select { color: #e8edf2; }
  :global(html.dark) .llm-model-select:hover { background: #2b3641; border-color: #3e4b58; }
  :global(html.dark) .llm-model-select option,
  :global(html.dark) .llm-model-select optgroup { background: #1b232b; color: #e8edf2; }
  
  .header-actions {
    display: flex;
    gap: 8px;
    align-items: center;
  }
  
  .refresh-btn,
  .close-btn {
    background: none;
    border: none;
    font-size: 20px;
    cursor: pointer;
    color: #666;
    line-height: 1;
    padding: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
  }
  
  .refresh-btn:hover,
  .close-btn:hover {
    background: #e0e0e0;
  }
  
  .prompt-input-section {
    display: flex;
    align-items: flex-start;
    gap: 8px;
    border-bottom: 1px solid #e0e0e0;
    margin-bottom: 12px;
    padding-bottom: 12px;
  }

  .query-input .llm-input {
    flex: 1;
    min-height: 32px;
    max-height: 80px;
    border: none;
    padding: 8px 12px;
    resize: vertical;
    font-family: inherit;
    font-size: 11px;
    outline: none;
    overflow-y: auto;
    scroll-behavior: smooth;
  }

  .query-input .llm-input::-webkit-scrollbar {
    width: 6px;
  }

  .query-input .llm-input::-webkit-scrollbar-track {
    background: #f8f9fa;
  }

  .query-input .llm-input::-webkit-scrollbar-thumb {
    background: #dee2e6;
    border-radius: 3px;
  }

  .query-input .llm-input::-webkit-scrollbar-thumb:hover {
    background: #c1c1c1;
  }
  
  .query-input {
    padding: 12px;
  }

  .llm-send-btn {
    width: 32px;
    height: 32px;
    min-width: 32px;
    padding: 0;
    border: none;
    border-radius: 50%;
    background: #007bff;
    color: white;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 16px;
  }

  .llm-send-btn::before {
    content: '→';
  }
  
  .llm-send-btn:hover:not(:disabled) {
    background: #0056b3;
  }
  
  .llm-send-btn:disabled {
    background: #ccc;
    cursor: not-allowed;
  }
  
  .llm-response {
    flex: 1 1 auto;
    /* a flex child will not shrink below its content without this, which is what
       pushed the answer out through the bottom of the panel */
    min-height: 0;
    max-height: none;
    /* the answer is untrusted third-party text, so it sets its own type size
       rather than inheriting the slide's, which is sized for a lecture hall */
    font-size: 13.5px;
    line-height: 1.55;
    overflow-wrap: anywhere;
    overscroll-behavior: contain;
    overflow-y: auto;
    padding: 12px;
    border-top: 1px solid #e0e0e0;
    scroll-behavior: smooth;
  }

  /* Custom scrollbar styling for webkit browsers */
  .llm-interface::-webkit-scrollbar,
  .llm-response::-webkit-scrollbar {
    width: 8px;
  }

  .llm-interface::-webkit-scrollbar-track,
  .llm-response::-webkit-scrollbar-track {
    background: #f1f1f1;
    border-radius: 4px;
  }

  .llm-interface::-webkit-scrollbar-thumb,
  .llm-response::-webkit-scrollbar-thumb {
    background: #c1c1c1;
    border-radius: 4px;
    transition: background 0.2s ease;
  }

  .llm-interface::-webkit-scrollbar-thumb:hover,
  .llm-response::-webkit-scrollbar-thumb:hover {
    background: #a1a1a1;
  }
  
  .loading {
    display: flex;
    align-items: center;
    gap: 12px;
    color: #666;
    font-weight: 500;
  }
  
  .loading-spinner {
    width: 20px;
    height: 20px;
    border: 2px solid #e0e0e0;
    border-top: 2px solid #007bff;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .error {
    color: #dc3545;
    font-weight: 500;
  }
  
  :deep(.response-content) {
    line-height: 1.6;
    color: #333;
  }

  /* Headings */
  :deep(.response-content) h1 {
    font-size: 1.8em;
    font-weight: 700;
    margin: 1em 0 0.5em;
    color: #1a202c;
    border-bottom: 2px solid #e2e8f0;
    padding-bottom: 0.3em;
  }

  :deep(.response-content) h2 {
    font-size: 1.5em;
    font-weight: 700;
    margin: 0.8em 0 0.4em;
    color: #2d3748;
  }

  :deep(.response-content) h3 {
    font-size: 1.25em;
    font-weight: 600;
    margin: 0.6em 0 0.3em;
    color: #2d3748;
  }

  :deep(.response-content) h4 {
    font-size: 1.1em;
    font-weight: 600;
    margin: 0.5em 0 0.25em;
    color: #4a5568;
  }

  /* Paragraphs */
  :deep(.response-content) p {
    margin: 0.8em 0;
  }

  /* Lists */
  :deep(.response-content) ul,
  :deep(.response-content) ol {
    margin: 0.8em 0;
    padding-left: 2em;
  }

  :deep(.response-content) li {
    margin: 0.4em 0;
  }

  :deep(.response-content) ul li {
    list-style-type: disc;
  }

  :deep(.response-content) ol li {
    list-style-type: decimal;
  }

  :deep(.response-content) li > ul,
  :deep(.response-content) li > ol {
    margin: 0.4em 0;
  }

  /* Emphasis */
  :deep(.response-content) strong {
    font-weight: 700;
    color: #1a202c;
  }

  :deep(.response-content) em {
    font-style: italic;
  }

  /* Links */
  :deep(.response-content) a {
    color: #3182ce;
    text-decoration: underline;
  }

  :deep(.response-content) a:hover {
    color: #2c5aa0;
  }

  /* Blockquotes */
  :deep(.response-content) blockquote {
    margin: 1em 0;
    padding: 0.8em 1em;
    border-left: 4px solid #3182ce;
    background: #f7fafc;
    color: #4a5568;
    font-style: italic;
  }

  :deep(.response-content) blockquote p {
    margin: 0;
  }

  /* Horizontal rule */
  :deep(.response-content) hr {
    margin: 1.5em 0;
    border: none;
    border-top: 2px solid #e2e8f0;
  }

  /* Tables */
  :deep(.response-content) table {
    width: 100%;
    border-collapse: collapse;
    margin: 1em 0;
    font-size: 0.9em;
  }

  :deep(.response-content) table th,
  :deep(.response-content) table td {
    padding: 0.6em 0.8em;
    border: 1px solid #e2e8f0;
    text-align: left;
  }

  :deep(.response-content) table th {
    background: #f7fafc;
    font-weight: 600;
    color: #2d3748;
  }

  :deep(.response-content) table tr:nth-child(even) {
    background: #f7fafc;
  }

  /* Inline code */
  :deep(.response-content) code {
    background: #f7fafc;
    padding: 0.2em 0.4em;
    border-radius: 4px;
    font-family: 'Courier New', Consolas, Monaco, monospace;
    font-size: 0.9em;
    color: #e53e3e;
    border: 1px solid #e2e8f0;
  }

  /* Code blocks */
  :deep(.response-content) pre {
    background: #2d3748;
    color: #e2e8f0;
    padding: 16px;
    border-radius: 6px;
    overflow-x: auto;
    margin: 12px 0;
    scroll-behavior: smooth;
  }

  :deep(.response-content) pre::-webkit-scrollbar {
    height: 6px;
  }

  :deep(.response-content) pre::-webkit-scrollbar-track {
    background: #1a202c;
    border-radius: 3px;
  }

  :deep(.response-content) pre::-webkit-scrollbar-thumb {
    background: #4a5568;
    border-radius: 3px;
  }

  :deep(.response-content) pre::-webkit-scrollbar-thumb:hover {
    background: #2d3748;
  }

  @media print {
    .llm-query-container {
      display: none !important;
    }
  }

  /* Add subtle shadows to indicate scrollable content */
  .llm-response::before {
    content: '';
    position: sticky;
    top: 0;
    display: block;
    height: 6px;
    background: linear-gradient(180deg, rgba(0,0,0,0.1) 0%, transparent 100%);
    margin: -12px -12px 8px -12px;
    opacity: 0;
    transition: opacity 0.2s ease;
  }

  .llm-response.has-scroll::before {
    opacity: 1;
  }

  
  /* a <code> inside a <pre> already sits in a styled block; without resetting the
     border and radius too it draws a second box inside the first */
  :deep(.response-content) pre code {
    background: none;
    padding: 0;
    color: inherit;
    border: 0;
    border-radius: 0;
    font-size: inherit;
  }
  </style>
