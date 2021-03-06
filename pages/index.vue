<template>
  <div id="wrapper">
		<div id="content-wrapper">
			<div id="intro">
				<div class="intro-section">
					<p class="intro-section-title">Browser Bookmarks Map</p>
					<p class="intro-section-text">Make an HTML 'map' of your Firefox or Google Chrome bookmarks.</p>
				</div>
				<div class="intro-section">
					<p class="intro-section-title">Features</p>
					<p class="intro-section-text">- Open in any browser.</p>
					<div class="intro-section-text">
						- <details>
							<summary>Simple accordion-style layout ...</summary>
							... to easily toggle content.
						</details>
					</div>
					<p class="intro-section-text">- Dark theme by default (easier on the eyes).</p>
					<p class="intro-section-text">- Convenient to share your bookmarks with others.</p>
					<p class="intro-section-text">- Your data is private and will never be recorded or shared.</p>
					<p class="intro-section-text">- A stylesheet is included so that you may adjust colors to your preference.</p>
				</div>
			</div>
			<div id="example">
				<img id="example-image" src="/img/bookmarks.jpg" alt="sample image">
				<div id="example-links">
					<a class="example-link" href="http://careful-turkey.surge.sh" target="_blank">
						<div class="example-link-icon-wrapper">
							<img class="example-link-icon" src="/icons/link.svg" alt="link">
						</div>
						<div class="example-link-text">Example</div>
					</a>
					<a class="example-link" href="https://github.com/xnhl/Bookmarks-Map-Website" target="_blank">
						<div class="example-link-icon-wrapper">
							<img class="example-link-icon" src="/icons/github.svg" alt="git hub">
						</div>
						<div class="example-link-text">GitHub</div>
					</a>
				</div>
			</div>
			<label id="label" for="input">
				<img id="label-icon" src="/icons/folder.svg" alt="select a file">
				<p id="label-text">Select a file</p>
			</label>
			<input id="input" type="file" name="input" @change="handleSelect">
			<p id="error" class="hide">Sorry, something went wrong. Try reloading the page or using another file.</p>
			<div id="output" :class="{ 'hide': !ready }">
				<div class="output-format" :class="{ 'ready': ready }" id="h" @click="save">
					<div class="output-format-icon-wrapper">
						<img class="output-format-icon" src="/icons/download.svg" alt="download">
					</div>
					<p class="output-format-text">HTML</p>
				</div>
				<div class="output-format" :class="{ 'ready': ready }" id="m" @click="save">
					<div class="output-format-icon-wrapper">
						<img class="output-format-icon" src="/icons/download.svg" alt="download">
					</div>
					<p class="output-format-text">Markdown</p>
				</div>
			</div>
			<div id="notes">
				<div id="notes-title">File Locations</div>
				<div class="note">- Firefox (Windows): <br/>Use the bookmarks menu to make a backup (.json)</div>
				<div class="note">- Chrome (Windows): <br/>C:\Users\USERNAME\AppData\Local\Google\Chrome\User Data\Default\Bookmarks (no file extension)</div>
			</div>
		</div>
  </div>
</template>

<script>
export default {
	data() {
		return {
			h: "",
			m: "",
			has_assets: false,
			ready: false
		}
	},
  methods: {
		save_assets: function() {
			if (!this.has_assets) {
				let FileSaver = require('file-saver')
				FileSaver.saveAs("/result_assets/bookmarks_styles.css", "bookmarks_styles.css")
				FileSaver.saveAs("/result_assets/OpenSans-Regular.ttf", "OpenSans-Regular.ttf")
				this.has_assets = true
			}
		},
		save: function(e) {
			let FileSaver = require('file-saver')
			if (e.target.closest(".output-format").id == "h") {
				let hypertext = new Blob([this.h], {type: "text/html;charset=utf-8"})
				FileSaver.saveAs(hypertext, "index.html")
				this.save_assets()
			} else if (e.target.closest(".output-format").id == "m") {
				let markdown = new Blob([this.m], {type: "text/markdown;charset=utf-8"})
				FileSaver.saveAs(markdown, "index.md")
				this.save_assets()
			}
		},
		handleSelect: function(e) {
			this.ready = false
			let label = document.getElementById("label")
			label.textContent = "Select a file"
			if (e.target.files.length) {
				let file = e.target.files[0]
				label.textContent = file.name
				let reader = new FileReader()
				reader.onload = () => {
					let parsed = JSON.parse(reader.result)
					if (parsed.roots) {
						this.convert_chrome(parsed)
					} else this.convert_firefox(parsed)
				}
				reader.onerror = () => {
					let err = document.getElementById("error")
					err.classList.remove("hide")
				}
				reader.readAsText(file)
			}
		},
		convert_chrome: function(input) {
			const mapData = (section) => {
				let children
				if (section.children && Array.isArray(section.children) && section.children.length) {
					children = section.children.map(child => mapData(child))
				} else children = undefined
				let type = section.type
				let name = section.name !== "" ? section.name.replace(/<|>/gi, "") : section.url
				let url = section.url ? section.url : undefined
				return {
					type,
					name,
					url,
					children
				}
			}
			let final_data = [
				mapData(input.roots.bookmark_bar),
				mapData(input.roots.other),
				mapData(input.roots.synced)
			]
			let elements = final_data.map(each => this.createElements(each)).join("")
			let elements_markdown = final_data.map(each => this.createElementsMarkdown(each)).join("")
			this.h = this.makeHTML(elements, "html")
			this.m = this.makeHTML(elements_markdown, "markdown")
			this.ready = true
		},
		convert_firefox: function(input) {
			const mapData = (section) => {
				let children
				if (section.children && Array.isArray(section.children) && section.children.length) {
					children = section.children.map(child => mapData(child))
				} else children = undefined
				let type = section.type == "text/x-moz-place-container" ? "folder" : (section.type == "text/x-moz-place" || section.type == "text/x-moz-place-separator") ? "url" : undefined
				let name = (section.title && section.title !== "") ? section.title.replace(/<|>/gi, "") : section.uri ? section.uri : "Untitled"
				let url = section.uri ? section.uri : "#"
				return {
					type,
					name,
					url,
					children
				}
			}
			let final_data = input.children.map(child => mapData(child))
			let elements = final_data.map(each => this.createElements(each)).join("")
			let elements_markdown = final_data.map(each => this.createElementsMarkdown(each)).join("")
			this.h = this.makeHTML(elements, "html")
			this.m = this.makeHTML(elements_markdown, "markdown")
			this.ready = true
		},
		createChildrenElements: function(each) {
			if (each.type == "folder") {
				let kids = []
				if (each.children && Array.isArray(each.children) && each.children.length) {
					kids = each.children.map(each => this.createChildrenElements(each)).join("")
				}
				return `\t<details class='child'><summary>${each.name !== "" ? each.name : each.url}</summary>\r\n${kids}</details>\r\n`
			} else if (each.type = "url") {
				return `\t\t<a href='${each.url}' target='_blank'><span class='link-title'>${each.name !== "" ? each.name : each.url}</span><span class='link-url'>${each.url}</span></a>\r\n`
			}
		},
		createElements: function(data) {
			let kids = []
			if (data.children && Array.isArray(data.children) && data.children.length) {
				kids = data.children.map(each => this.createChildrenElements(each)).join("")
			}
			return `<details open><summary>${data.name}</summary>\r\n${kids}</details>\r\n`
		},
		createChildrenElementsMarkdown: function(each) {
			if (each.type == "folder") {
				let kids = []
				if (each.children && Array.isArray(each.children) && each.children.length) {
					kids = each.children.map(each => this.createChildrenElementsMarkdown(each)).join("")
				}
				return `\t<details class='child'><summary>${each.name !== "" ? each.name : each.url}</summary>\r\n${kids}</details>\r\n`
			} else if (each.type = "url") {
				return `\t\t<a href='${each.url}' target='_blank'>${each.name !== "" ? each.name : each.url}</a>\r\n`
			}
		},
		createElementsMarkdown: function(data) {
			let kids = []
			if (data.children && Array.isArray(data.children) && data.children.length) {
				kids = data.children.map(each => this.createChildrenElementsMarkdown(each)).join("")
			}
			return `<details open><summary>${data.name}</summary>\r\n${kids}</details>\r\n`
		},
		makeHTML: function(html, type) {
			let h = `<!DOCTYPE html>
<html lang='en'>
	<head>
		<meta charset='UTF-8'>
		<meta name='viewport' content='width=device-width, initial-scale=1.0'>
		<title>Bookmarks</title>
		<link rel='stylesheet' href='./bookmarks_styles.css'>
	</head>
	<body>
		${html}</body>
</html>`
			let m = `<html lang='en'>
	<head>
		<meta charset='UTF-8'>
		<meta name='viewport' content='width=device-width, initial-scale=1.0'>
		<title>Bookmarks</title>
		<link rel='stylesheet' href='./bookmarks_styles.css'>
	</head>
	<body>
		${html}</body>
</html>`
			return type == "html" ? h : type == "markdown" ? m : ""
		}
	}
}
</script>

<style lang="sass">
#wrapper
	@include pageWrapper
	#content-wrapper
		margin: 0 auto
		padding: 0.5rem
		max-width: 50rem
		background: #ddd
		@include flexCenter
		border-radius: 0.5rem
		flex-direction: column
		border-radius: 1rem/50rem
		transition: all 0.2s ease-in-out
		box-shadow: 0 0 0.25rem 0.15rem rgba(black, 0.1)
		@media (min-width: 50rem)
			margin: calc(0.5rem + 2vw) auto
		#example
			@include flexCenter
			flex-direction: column
			#example-image
				height: auto
				max-width: 100%
				border-radius: 0.25rem
				box-sizing: content-box
			#example-links
				width: 100%
				margin: 0.5rem auto
				max-width: 20rem
				@include flexCenter
				align-items: stretch
				.example-link
					flex: 1
					margin: 0.5rem
					cursor: pointer
					background: #eee
					@include flexCenter
					text-decoration: none
					border-radius: 1rem/10rem
					transition: all 0.2s ease-in-out
					box-shadow: 0 0 0.25rem 0.15rem rgba(black, 0.05)
					&:hover
						box-shadow: 0 0 0.25rem 0.15rem rgba(black, 0.15)
						.example-link-icon-wrapper
							.example-link-icon
								transform: scale(1.25)
					.example-link-icon-wrapper
						.example-link-icon
							width: 1.5rem
							height: 1.5rem
							padding: 0.5rem
							box-sizing: content-box
							transition: all 0.1s ease-in-out
					.example-link-text
						color: black
		#intro
			margin-bottom: 2rem
			max-width: 45rem
			@include flexCenter
			.intro-section
				margin: 0.5rem
				@include flexCenter
				flex-direction: column
				.intro-section-title
					padding: 0.5rem
					font-size: 1.5rem
					text-align: center
					line-height: 1.5rem
					@include flexCenter
				.intro-section-text
					padding: 0.25rem
					font-size: 1.1rem
					text-align: center
					line-height: 1.1rem
					text-indent: 0.5rem
					@include flexCenter
					details
						summary
							outline: none
		#label
			margin: 2rem auto 1rem auto
			padding: 1.5rem
			cursor: pointer
			background: #eee
			font-size: 1.5rem
			line-height: 1.5rem
			@include flexCenter
			border-radius: 0.5rem
			border-radius: 1rem/10rem
			transition: all 0.2s ease-in-out
			box-shadow: 0 0 0.25rem 0.15rem rgba(black, 0.1)
			&:hover
				box-shadow: 0 0 0.25rem 0.2rem rgba(black, 0.15)
				#label-icon
					transform: scale(1.25)
			#label-text
				padding: 0.5rem
				font-size: 1.5rem
				line-height: 1.5rem
			#label-icon
				width: 1.75rem
				height: 1.75rem
				padding: 0.5rem
				box-sizing: content-box
				transition: all 0.1s ease-in-out
		#notes
			@include flexCenter
			flex-direction: column
			#notes-title
				padding: 0.5rem
				font-size: 1.5rem
				text-align: center
				line-height: 1.5rem
				@include flexCenter
			.note
				max-width: 45rem
				padding: 0.25rem
				font-size: 1.1rem
				text-align: center
				line-height: 1.1rem
				@include flexCenter
		#input
			display: none
		#output
			width: 100%
			max-width: 30rem
			margin: 2rem auto
			@include flexCenter
			.output-format
				width: 100%
				padding: 1rem
				margin: 0.5rem
				cursor: pointer
				background: #eee
				@include flexCenter
				border-radius: 0.25rem
				border-radius: 1rem/5rem
				transition: all 0.2s ease-in-out
				box-shadow: 0 0 0.25rem 0.15rem rgba(green, 0.1)
				@media (min-width: 30rem)
					flex: 1
				&.ready
					background: rgba(green, 0.25)
				&:hover
					box-shadow: 0 0 0.25rem 0.15rem rgba(green, 0.15)
					background: rgba(green, 0.33)
					.output-format-icon-wrapper
						.output-format-icon
							transform: scale(1.25)
				.output-format-text
					font-size: 1.5rem
					text-align: center
					line-height: 1.5rem
				.output-format-icon-wrapper
					.output-format-icon
						width: 1.5rem
						height: 1.5rem
						padding: 0.5rem
						box-sizing: content-box
						transition: all 0.1s ease-in-out
</style>
