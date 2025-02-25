
<script>
import commonMixins from '../mixins.js'
import Prism from "prismjs"
import Tags from "bootstrap5-tags"
import Attachments from './Attachments.vue'
import Headers from './Headers.vue'

export default {
	props: {
		message: Object,
		existingTags: Array
	},

	components: {
		Attachments,
		Headers,
	},

	mixins: [commonMixins],

	data() {
		return {
			srcURI: false,
			iframes: [], // for resizing
			showTags: false, // to force re-rendering of component
			canSaveTags: false, // prevent auto-saving tags on render
			messageTags: [],
			allTags: [],
			loadHeaders: false,
			showMobileButtons: false,
			scaleHTMLPreview: 'display',
			// keys names match bootstrap icon names 
			responsiveSizes: {
				phone: 'width: 322px; height: 570px',
				tablet: 'width: 768px; height: 1024px',
				display: 'width: 100%; height: 100%',
			},
		}
	},

	watch: {
		// handle changes to the URL messageID
		message: {
			handler() {
				let self = this
				self.showTags = false
				self.canSaveTags = false
				self.messageTags = self.message.Tags
				self.allTags = self.existingTags
				self.loadHeaders = false
				self.scaleHTMLPreview = 'display' // default view
				// delay to select first tab and add HTML highlighting (prev/next)
				self.$nextTick(function () {
					self.renderUI()
					self.showTags = true
					self.$nextTick(function () {
						Tags.init("select[multiple]")
						window.setTimeout(function () {
							self.canSaveTags = true
						}, 200)
					})
				})
			},
			// force eager callback execution
			immediate: true
		},
		messageTags() {
			// save changes to tags
			if (this.canSaveTags) {
				this.saveTags()
			}
		},
		scaleHTMLPreview() {
			if (this.scaleHTMLPreview == 'display') {
				let self = this
				window.setTimeout(function () {
					self.resizeIframes()
				}, 500)
			}
		}
	},

	mounted() {
		let self = this
		self.showTags = false
		self.canSaveTags = false
		self.allTags = self.existingTags
		window.addEventListener("resize", self.resizeIframes)
		self.renderUI()

		let headersTab = document.getElementById('nav-headers-tab')
		headersTab.addEventListener('shown.bs.tab', function (event) {
			self.loadHeaders = true
		})

		let rawTab = document.getElementById('nav-raw-tab')
		rawTab.addEventListener('shown.bs.tab', function (event) {
			self.srcURI = 'api/v1/message/' + self.message.ID + '/raw'
			self.resizeIframes()
		})

		self.showTags = true
		self.$nextTick(function () {
			self.$nextTick(function () {
				Tags.init('select[multiple]')
				window.setTimeout(function () {
					self.canSaveTags = true
				}, 200)
			})
		})
	},

	unmounted: function () {
		window.removeEventListener("resize", this.resizeIframes)
	},

	methods: {
		isHTMLTabSelected: function () {
			this.showMobileButtons = this.$refs.navhtml
				&& this.$refs.navhtml.classList.contains('active')
		},
		renderUI: function () {
			let self = this
			// click the first non-disabled tab
			document.querySelector('#nav-tab button:not([disabled])').click()
			document.activeElement.blur() // blur focus
			document.getElementById('message-view').scrollTop = 0

			self.isHTMLTabSelected()

			document.querySelectorAll('button[data-bs-toggle="tab"]').forEach(function (listObj) {
				listObj.addEventListener('shown.bs.tab', function (event) {
					self.isHTMLTabSelected()
				})
			})

			// delay 0.2s until vue has rendered the iframe content
			window.setTimeout(function () {
				let p = document.getElementById('preview-html')
				if (p) {
					// make links open in new window
					let anchorEls = p.contentWindow.document.body.querySelectorAll('a')
					for (var i = 0; i < anchorEls.length; i++) {
						let anchorEl = anchorEls[i]
						let href = anchorEl.getAttribute('href')

						if (href && href.match(/^http/)) {
							anchorEl.setAttribute('target', '_blank')
						}
					}
					self.resizeIframes()
				}
			}, 200)

			// html highlighting
			window.Prism = window.Prism || {}
			window.Prism.manual = true
			Prism.highlightAll()
		},

		resizeIframe: function (el) {
			let i = el.target
			i.style.height = i.contentWindow.document.body.scrollHeight + 50 + 'px'
		},

		resizeIframes: function () {
			if (this.scaleHTMLPreview != 'display') {
				return
			}
			let h = document.getElementById('preview-html')
			if (h) {
				h.style.height = h.contentWindow.document.body.scrollHeight + 50 + 'px'
			}

		},

		// set the iframe body & text colors based on current theme
		initRawIframe: function (el) {
			let bodyStyles = window.getComputedStyle(document.body, null)
			let bg = bodyStyles.getPropertyValue('background-color')
			let txt = bodyStyles.getPropertyValue('color')

			let body = el.target.contentWindow.document.querySelector('body')
			if (body) {
				body.style.color = txt
				body.style.backgroundColor = bg
			}

			this.resizeIframe(el)
		},

		saveTags: function () {
			let self = this

			var data = {
				ids: [this.message.ID],
				tags: this.messageTags
			}

			self.put('api/v1/tags', data, function (response) {
				self.scrollInPlace = true
				self.$emit('loadMessages')
			})
		},

		// Convert plain text to HTML including anchor links
		textToHTML: function (s) {
			let html = s

			// full links with http(s)
			let re = /(\b(https?|ftp):\/\/[\-\w@:%_\+.~#?,&\/\/=;]+)\b/gim
			html = html.replace(re, '˱˱˱a href=ˠˠˠ$&ˠˠˠ target=_blank rel=noopener˲˲˲$&˱˱˱/a˲˲˲')

			// plain www links without https?:// prefix
			let re2 = /(^|[^\/])(www\.[\S]+(\b|$))/gim
			html = html.replace(re2, '$1˱˱˱a href=ˠˠˠhttp://$2ˠˠˠ target=ˠˠˠ_blankˠˠˠ rel=ˠˠˠnoopenerˠˠˠ˲˲˲$2˱˱˱/a˲˲˲')

			// escape to HTML & convert <>" back
			html = html
				.replace(/&/g, "&amp;")
				.replace(/</g, "&lt;")
				.replace(/>/g, "&gt;")
				.replace(/"/g, "&quot;")
				.replace(/'/g, "&#039;")
				.replace(/˱˱˱/g, '<')
				.replace(/˲˲˲/g, '>')
				.replace(/ˠˠˠ/g, '"')

			return html
		}
	}
}
</script>

<template>
	<div v-if="message" id="message-view" class="px-2 px-md-0 mh-100" style="overflow-y: scroll;">
		<div class="row w-100">
			<div class="col-md">
				<table class="messageHeaders">
					<tbody>
						<tr>
							<th class="small">From</th>
							<td class="privacy">
								<span v-if="message.From">
									<span v-if="message.From.Name">{{ message.From.Name + " " }}</span>
									<span v-if="message.From.Address" class="small">
										&lt;{{ message.From.Address }}&gt;
									</span>
								</span>
								<span v-else>
									[ Unknown ]
								</span>
							</td>
						</tr>
						<tr class="small">
							<th>To</th>
							<td class="privacy">
								<span v-if="message.To && message.To.length" v-for="(t, i) in message.To">
									<template v-if="i > 0">, </template>
									<span class="text-nowrap">{{ t.Name + " &lt;" + t.Address + "&gt;" }}</span>
								</span>
								<span v-else class="text-body-secondary">[Undisclosed recipients]</span>
							</td>
						</tr>
						<tr v-if="message.Cc && message.Cc.length" class="small">
							<th>Cc</th>
							<td class="privacy">
								<span v-for="(t, i) in message.Cc">
									<template v-if="i > 0">,</template>
									{{ t.Name + " &lt;" + t.Address + "&gt;" }} </span>
							</td>
						</tr>
						<tr v-if="message.Bcc && message.Bcc.length" class="small">
							<th>Bcc</th>
							<td class="privacy">
								<span v-for="(t, i) in message.Bcc">
									<template v-if="i > 0">,</template>
									{{ t.Name + " &lt;" + t.Address + "&gt;" }}
								</span>
							</td>
						</tr>
						<tr v-if="message.ReplyTo && message.ReplyTo.length" class="small">
							<th class="text-nowrap">Reply-To</th>
							<td class="privacy text-body-secondary">
								<span v-for="(t, i) in message.ReplyTo">
									<template v-if="i > 0">,</template>
									{{ t.Name + " &lt;" + t.Address + "&gt;" }} </span>
							</td>
						</tr>
						<tr v-if="message.ReturnPath && message.ReturnPath != message.From.Address" class="small">
							<th class="text-nowrap">Return-Path</th>
							<td class="privacy text-body-secondary">&lt;{{ message.ReturnPath }}&gt;</td>
						</tr>
						<tr>
							<th class="small">Subject</th>
							<td>
								<strong v-if="message.Subject != ''">{{ message.Subject }}</strong>
								<small class="text-body-secondary" v-else>[ no subject ]</small>
							</td>
						</tr>
						<tr class="d-md-none small">
							<th class="small">Date</th>
							<td>{{ messageDate(message.Date) }}</td>
						</tr>

						<tr class="small" v-if="showTags">
							<th>Tags</th>
							<td>
								<select class="form-select small tag-selector" v-model="messageTags" multiple
									data-full-width="false" data-suggestions-threshold="1" data-allow-new="true"
									data-clear-end="true" data-allow-clear="true" data-placeholder="Add tags..."
									data-badge-style="secondary" data-regex="^([a-zA-Z0-9\-\ \_]){3,}$"
									data-separator="|,|">
									<option value="">Type a tag...</option>
									<!-- you need at least one option with the placeholder -->
									<option v-for="t in allTags" :value="t">{{ t }}</option>
								</select>
								<div class="invalid-feedback">Please select a valid tag.</div>
							</td>
						</tr>
					</tbody>
				</table>
			</div>
			<div class="col-md-auto d-none d-md-block text-end mt-md-3">
				<div class="mt-2 mt-md-0" v-if="allAttachments(message)">
					<span class="badge rounded-pill text-bg-secondary p-2">
						Attachment<span v-if="allAttachments(message).length > 1">s</span>
						({{ allAttachments(message).length }})
					</span>
				</div>
			</div>
		</div>

		<nav>
			<div class="nav nav-tabs my-3" id="nav-tab" role="tablist">
				<button class="nav-link" id="nav-html-tab" data-bs-toggle="tab" data-bs-target="#nav-html" type="button"
					role="tab" aria-controls="nav-html" aria-selected="true" v-if="message.HTML" ref="navhtml"
					v-on:click="resizeIframes()">HTML</button>
				<button class="nav-link" id="nav-html-source-tab" data-bs-toggle="tab" data-bs-target="#nav-html-source"
					type="button" role="tab" aria-controls="nav-html-source" aria-selected="false" v-if="message.HTML">
					HTML <span class="d-sm-none">Src</span><span class="d-none d-sm-inline">Source</span>
				</button>
				<button class="nav-link" id="nav-plain-text-tab" data-bs-toggle="tab" data-bs-target="#nav-plain-text"
					type="button" role="tab" aria-controls="nav-plain-text" aria-selected="false"
					:class="message.HTML == '' ? 'show' : ''">
					Text
				</button>
				<button class="nav-link" id="nav-headers-tab" data-bs-toggle="tab" data-bs-target="#nav-headers"
					type="button" role="tab" aria-controls="nav-headers" aria-selected="false">
					<span class="d-sm-none">Hdrs</span><span class="d-none d-sm-inline">Headers</span>
				</button>
				<button class="nav-link" id="nav-raw-tab" data-bs-toggle="tab" data-bs-target="#nav-raw" type="button"
					role="tab" aria-controls="nav-raw" aria-selected="false">
					Raw
				</button>

				<div class="d-none d-lg-block ms-auto me-2" v-if="showMobileButtons">
					<template v-for="vals, key in responsiveSizes">
						<button class="btn" :disabled="scaleHTMLPreview == key" :title="'Switch to ' + key + ' view'"
							v-on:click="scaleHTMLPreview = key">
							<i class="bi" :class="'bi-' + key"></i>
						</button>
					</template>
				</div>
			</div>
		</nav>

		<div class="tab-content mb-5" id="nav-tabContent">
			<div v-if="message.HTML != ''" class="tab-pane fade show" id="nav-html" role="tabpanel"
				aria-labelledby="nav-html-tab" tabindex="0">
				<div id="responsive-view" :class="scaleHTMLPreview" :style="responsiveSizes[scaleHTMLPreview]">
					<iframe target-blank="" class="tab-pane d-block" id="preview-html" :srcdoc="message.HTML"
						v-on:load="resizeIframe" frameborder="0" style="width: 100%; height: 100%;">
					</iframe>
				</div>
				<Attachments v-if="allAttachments(message).length" :message="message"
					:attachments="allAttachments(message)"></Attachments>
			</div>
			<div class="tab-pane fade" id="nav-html-source" role="tabpanel" aria-labelledby="nav-html-source-tab"
				tabindex="0" v-if="message.HTML">
				<pre><code class="language-html">{{ message.HTML }}</code></pre>
			</div>
			<div class="tab-pane fade" id="nav-plain-text" role="tabpanel" aria-labelledby="nav-plain-text-tab" tabindex="0"
				:class="message.HTML == '' ? 'show' : ''">
				<div class="text-view" v-html="textToHTML(message.Text)"></div>
				<Attachments v-if="allAttachments(message).length" :message="message"
					:attachments="allAttachments(message)"></Attachments>
			</div>
			<div class="tab-pane fade" id="nav-headers" role="tabpanel" aria-labelledby="nav-headers-tab" tabindex="0">
				<Headers v-if="loadHeaders" :message="message"></Headers>
			</div>
			<div class="tab-pane fade" id="nav-raw" role="tabpanel" aria-labelledby="nav-raw-tab" tabindex="0">
				<iframe v-if="srcURI" :src="srcURI" v-on:load="initRawIframe" frameborder="0"
					style="width: 100%; height: 300px"></iframe>
			</div>
		</div>
	</div>
</template>
