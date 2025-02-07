<script lang="ts">
	import { scale } from "svelte/transition"
	import { flip } from "svelte/animate"

	import SortableList from "svelte-sortable-list"
	import json5 from "json5"
	import { Button, CodeSnippet, InlineNotification, Modal, Search } from "carbon-components-svelte"

	import { getAllMods, getConfig, mergeConfig, getManifestFromModID, modIsFramework, getModFolder, sortMods } from "$lib/utils"
	import Mod from "$lib/Mod.svelte"
	import TextInputModal from "$lib/TextInputModal.svelte"
	import { goto } from "$app/navigation"

	import Add from "carbon-icons-svelte/lib/Add.svelte"
	import AddAlt from "carbon-icons-svelte/lib/AddAlt.svelte"
	import SubtractAlt from "carbon-icons-svelte/lib/SubtractAlt.svelte"
	import Rocket from "carbon-icons-svelte/lib/Rocket.svelte"
	import Settings from "carbon-icons-svelte/lib/Settings.svelte"
	import TrashCan from "carbon-icons-svelte/lib/TrashCan.svelte"
	import Close from "carbon-icons-svelte/lib/Close.svelte"
	import CloudUpload from "carbon-icons-svelte/lib/CloudUpload.svelte"
	import Filter from "carbon-icons-svelte/lib/Filter.svelte"
	import { OptionType } from "../../../src/types"

	let enabledMods: { value: string }[] = [],
		disabledMods: { value: string }[] = []

	let deleteModModalOpen = false
	let deleteModInProgress: string

	let forceModListsUpdate: number = Math.random()

	$: enabledMods = getConfig().loadOrder.map((a) => {
		return { value: a, dummy: forceModListsUpdate }
	})

	$: disabledMods = getAllMods()
		.filter((a) => !getConfig().loadOrder.includes(a))
		.sort((a, b) => (modIsFramework(a) ? getManifestFromModID(a).name : a).localeCompare(modIsFramework(b) ? getManifestFromModID(b).name : b, undefined, { numeric: true, sensitivity: "base" }))
		.map((a) => {
			return { value: a, dummy: forceModListsUpdate }
		})

	let changed = false

	let dependencyCycleModalOpen = false
	let frameworkDeployModalOpen = false
	let deployOutput = ""
	let deployFinished = false

	window.ipc.receive("frameworkDeployModalOpen", () => {
		frameworkDeployModalOpen = true
	})

	window.ipc.receive("frameworkDeployOutput", (output: string) => {
		deployOutput = output
		setTimeout(() => {
			document.getElementById("deployOutputCodeElement")?.scrollIntoView({
				behavior: "smooth",
				block: "end"
			})
		}, 100)
	})

	window.ipc.receive("frameworkDeployFinished", () => {
		deployFinished = true
	})

	let modNameInputModal: TextInputModal
	let modNameInputModalOpen = false

	let modChunkInputModal: TextInputModal
	let modChunkInputModalOpen = false

	let rpkgModExtractionInProgress = false
	let frameworkModExtractionInProgress = false

	let invalidFrameworkZipModalOpen = false

	let invalidModModalOpen = false

	let modFilePath = ""

	let rpkgModName: string
	let rpkgModChunk: string

	let frameworkModScriptsWarningOpen = false

	async function addMod() {
		window.ipc.send("modFileOpenDialog")

		window.ipc.receive("modFileOpenDialogResult", (modFilePopupResult: string[] | undefined) => {
			if (!modFilePopupResult) {
				return
			}

			modFilePath = modFilePopupResult[0]

			if (modFilePath.endsWith(".rpkg")) {
				modNameInputModalOpen = true
			} else {
				window.fs.emptyDirSync("./staging")

				window.child_process.execSync(`"..\\Third-Party\\7z.exe" x "${modFilePath}" -aoa -y -o"./staging"`)

				if (window.fs.readdirSync("./staging").length == 1 && window.fs.readdirSync("./staging").every((a) => a.endsWith(".rpkg"))) {
					modFilePath = window.path.resolve(window.path.join("./staging", window.fs.readdirSync("./staging")[0]))
					modNameInputModalOpen = true
				} else {
					// framework mod

					frameworkModExtractionInProgress = true

					if (window.klaw("./staging", { depthLimit: 0, nodir: true }).length) {
						frameworkModExtractionInProgress = false
						invalidFrameworkZipModalOpen = true
						return
					}

					if (!window.fs.readdirSync("./staging").every((a) => window.fs.existsSync(window.path.join("./staging", a, "manifest.json")))) {
						frameworkModExtractionInProgress = false
						invalidModModalOpen = true
						return
					}

					try {
						window.fs.readdirSync("./staging").forEach((a) => json5.parse(window.fs.readFileSync(window.path.join("./staging", a, "manifest.json"), "utf8")))
					} catch {
						frameworkModExtractionInProgress = false
						invalidModModalOpen = true
						return
					}

					if (
						window.fs
							.readdirSync("./staging")
							.some(
								(a) =>
									json5.parse(window.fs.readFileSync(window.path.join("./staging", a, "manifest.json"), "utf8")).scripts ||
									json5.parse(window.fs.readFileSync(window.path.join("./staging", a, "manifest.json"), "utf8")).options?.some((b) => b.scripts)
							)
					) {
						frameworkModExtractionInProgress = false

						frameworkModScriptsWarningOpen = true
					} else {
						window.fs.copySync("./staging", "../Mods")

						window.originalFs.writeFileSync(window.path.join("..", "Mods", window.fs.readdirSync("./staging")[0], "manifest.json:SMFExtractionTag"), "Extracted via SMF")

						window.fs.removeSync("./staging")

						window.location.reload()

						frameworkModExtractionInProgress = false
					}
				}
			}
		})
	}

	async function installRPKGMod() {
		rpkgModExtractionInProgress = true

		window.fs.ensureDirSync(window.path.join("..", "Mods", rpkgModName, rpkgModChunk))
		window.fs.copyFileSync(modFilePath, window.path.join("..", "Mods", rpkgModName, rpkgModChunk, window.path.basename(modFilePath)))

		window.fs.removeSync("./staging")

		window.location.reload()

		rpkgModExtractionInProgress = false
	}

	let displayExtractedModsDialog = false
	const extractedMods: string[] = []

	if (!getConfig().developerMode) {
		// If no mods have the tag (likely updated from older SMF)
		if (getAllMods().filter((a) => modIsFramework(a)).every((a) => !window.originalFs.existsSync(window.path.join(getModFolder(a), "manifest.json:SMFExtractionTag")))) {
			for (const mod of getAllMods().filter((a) => modIsFramework(a))) {
				const modFolder = getModFolder(mod)

				// Assume every mod has been installed correctly
				window.originalFs.writeFileSync(window.path.join(modFolder, "manifest.json:SMFExtractionTag"), "Extracted via SMF")
			}
		}

		for (const mod of getAllMods().filter((a) => modIsFramework(a))) {
			const modFolder = getModFolder(mod)

			if (!window.originalFs.existsSync(window.path.join(modFolder, "manifest.json:SMFExtractionTag"))) {
				extractedMods.push(getManifestFromModID(mod).name)
				displayExtractedModsDialog = true
				window.originalFs.writeFileSync(window.path.join(modFolder, "manifest.json:SMFExtractionTag"), "Extracted via SMF") // Will prevent the message from being shown again for the same mod
			}
		}
	}

	let uploadedLogURL = ""
	let uploadLogModalOpen = false
	let uploadLogFailedModalOpen = false

	let availableModFilter = ""
	let enabledModFilter = ""
</script>

<div class="grid grid-cols-2 gap-4 w-full mb-16">
	<div class="w-full">
		<div class="flex gap-4 items-center justify-center" transition:scale>
			<h1 class="flex-grow">Available Mods</h1>
			<div>
				<Search icon={Filter} placeholder="Filter available mods" bind:value={availableModFilter} />
			</div>
			<Button
				kind="primary"
				icon={Add}
				on:click={() => {
					addMod()
				}}
			>
				Add a Mod
			</Button>
		</div>
		<br />
		<div class="h-[90vh] overflow-y-auto">
			{#each disabledMods.filter((a) => ((modIsFramework(a.value) ? getManifestFromModID(a.value).name : a.value) + (modIsFramework(a.value) ? getManifestFromModID(a.value).description : "")).toLowerCase().includes(availableModFilter.toLowerCase())) as item (item.value)}
				<div animate:flip={{ duration: 300 }}>
					<div transition:scale>
						<Mod
							isFrameworkMod={modIsFramework(item.value)}
							manifest={modIsFramework(item.value) ? getManifestFromModID(item.value) : undefined}
							rpkgModName={!modIsFramework(item.value) ? item.value : undefined}
						>
							<Button
								kind="primary"
								icon={AddAlt}
								on:click={() => {
									mergeConfig({
										loadOrder: [...getConfig().loadOrder, item.value]
									})
									changed = true
									forceModListsUpdate = Math.random()
								}}
							>
								Enable
							</Button>
							<Button
								kind="danger"
								icon={TrashCan}
								on:click={() => {
									deleteModInProgress = item.value
									deleteModModalOpen = true
								}}
							>
								Delete
							</Button>
						</Mod>
					</div>
					<br />
				</div>
			{/each}
		</div>
	</div>
	<div class="w-full">
		<div class="flex gap-4 items-center justify-center" transition:scale>
			<h1 class="flex-grow">{changed && !deployFinished ? "To Be Applied" : "Enabled Mods"}</h1>
			<div>
				<Search icon={Filter} placeholder="Filter enabled mods" bind:value={enabledModFilter} />
			</div>
			<Button
				kind="primary"
				style={changed && !deployFinished ? "background-color: green" : ""}
				icon={Rocket}
				on:click={() => {
					if (sortMods()) {
						deployOutput = ""
						deployFinished = false
						window.ipc.send("deploy")
					} else {
						dependencyCycleModalOpen = true
					}
				}}
			>
				Apply
			</Button>
		</div>
		<br />
		<div class="h-[90vh] overflow-y-auto">
			<SortableList
				list={enabledMods}
				key="value"
				on:sort={(event) => {
					mergeConfig({
						loadOrder: event.detail.map((a) => a.value)
					})
					forceModListsUpdate = Math.random()
					changed = true
				}}
				let:item
			>
				<div class="cursor-grab">
					<Mod
						isFrameworkMod={modIsFramework(item.value)}
						manifest={modIsFramework(item.value) ? getManifestFromModID(item.value) : undefined}
						rpkgModName={!modIsFramework(item.value) ? item.value : undefined}
						darken={!((modIsFramework(item.value) ? getManifestFromModID(item.value).name : item.value) + (modIsFramework(item.value) ? getManifestFromModID(item.value).description : "")).toLowerCase().includes(enabledModFilter.toLowerCase())}
					>
						{#if modIsFramework(item.value) && getManifestFromModID(item.value)?.options?.filter((a) => a.type != OptionType.conditional)?.length}
							<Button
								kind="ghost"
								icon={Settings}
								iconDescription="Adjust this mod's settings"
								on:click={() => {
									goto(`/settings?mod=${getManifestFromModID(item.value).id}`)
								}}
							/>
						{/if}
						<Button
							kind="danger"
							icon={SubtractAlt}
							on:click={() => {
								mergeConfig({
									loadOrder: getConfig().loadOrder.filter((a) => a != item.value)
								})
								changed = true
								forceModListsUpdate = Math.random()
							}}
						>
							Disable
						</Button>
					</Mod>
					<br />
				</div>
			</SortableList>
		</div>
	</div>
</div>

<Modal
	danger
	bind:open={deleteModModalOpen}
	modalHeading="Delete mod"
	primaryButtonText="Delete the mod"
	secondaryButtonText="Cancel"
	on:click:button--secondary={() => (deleteModModalOpen = false)}
	on:submit={() => {
		window.fs.removeSync(getModFolder(deleteModInProgress))
		deleteModModalOpen = false
		window.location.reload()
	}}
	shouldSubmitOnEnter={false}
>
	<p>
		{#if deleteModInProgress}
			Are you sure you want to permanently remove the <i>{modIsFramework(deleteModInProgress) ? getManifestFromModID(deleteModInProgress).name : deleteModInProgress}</i>
			mod from the Mods folder? You cannot undo this.
		{/if}
	</p>
</Modal>

<Modal alert bind:open={dependencyCycleModalOpen} modalHeading="Dependency cycle (couldn't sort mods)" primaryButtonText="OK" shouldSubmitOnEnter={false}>
	<p>The framework couldn't sort your mods! Ask the developer of whichever mod you most recently installed to investigate this. Also, report this to Atampy26 on Hitman Forum or Discord.</p>
</Modal>

<Modal passiveModal open={frameworkDeployModalOpen} modalHeading="Applying your mods" preventCloseOnClickOutside>
	Your mods are being deployed. This may take a while - grab a coffee or something.
	<br />
	<pre class="mt-2 h-[10vh] overflow-y-auto whitespace-pre-wrap"><code id="deployOutputCodeElement">{deployOutput}</code></pre>
	{#if deployOutput.split("\n").some((a) => a.startsWith("WARN")) || deployOutput.split("\n").some((a) => a.startsWith("ERROR"))}
		<br />
		<div class="flex flex-row gap-2 flex-wrap max-h-[15vh] overflow-y-auto">
			{#each deployOutput.split("\n").filter((a) => a.startsWith("WARN") || a.startsWith("ERROR")) as line}
				<InlineNotification hideCloseButton lowContrast kind={line.startsWith("WARN") ? "warning" : "error"}>
					<div slot="title" class="-mt-1 text-lg">
						{line.startsWith("WARN") ? "Warning" : "Error"}
					</div>
					<div slot="subtitle">{line.replace("WARN ", "").replace("ERROR ", "")}</div>
				</InlineNotification>
			{/each}
		</div>
	{/if}

	{#if deployFinished}
		<br />
		<div class="flex gap-4 items-center">
			{#if deployOutput.includes("Deployed all mods successfully.") && !deployOutput.split("\n").some((a) => a.startsWith("WARN"))}
				<Button kind="primary" icon={Close} on:click={() => (frameworkDeployModalOpen = false)}>Close</Button>
				<span class="text-green-300">Deploy successful</span>
			{:else if deployOutput.includes("Deployed all mods successfully.") && deployOutput.split("\n").some((a) => a.startsWith("WARN"))}
				<Button kind="primary" icon={Close} on:click={() => (frameworkDeployModalOpen = false)}>Close</Button>
				<span class="text-yellow-300">Potential issues in deployment</span>
			{:else}
				<Button kind="primary" icon={Close} on:click={() => (frameworkDeployModalOpen = false)}>Close</Button>
				<Button
					kind="primary"
					icon={CloudUpload}
					on:click={async () => {
						const req = await fetch("http://hitman-resources.netlify.app/.netlify/functions/upload-smf-log", {
							method: "POST",
							headers: {
								"Content-Type": "application/json"
							},
							body: JSON.stringify({ content: "Config:\n" + JSON.stringify(getConfig()) + "\n\nDeploy log:\n" + deployOutput })
						})

						if (req.status == 200) {
							uploadedLogURL = await req.text()

							frameworkDeployModalOpen = false
							uploadLogModalOpen = true
						} else {
							uploadLogFailedModalOpen = true
						}
					}}
				>
					Upload mod list and log
				</Button>
				<span class="text-red-300">Deploy unsuccessful</span>
			{/if}
		</div>
	{/if}
</Modal>

<TextInputModal
	bind:this={modNameInputModal}
	showingModal={modNameInputModalOpen}
	modalText="Mod name"
	modalPlaceholder="Amazing RPKG Mod"
	on:close={() => {
		rpkgModName = modNameInputModal.value

		try {
			var result = [...modFilePath.matchAll(/(chunk[0-9]*(?:patch.*)?)\.rpkg/g)]
			result = [...result[result.length - 1][result[result.length - 1].length - 1].matchAll(/(chunk[0-9]*)/g)]
			rpkgModChunk = result[result.length - 1][result[result.length - 1].length - 1]

			if (!rpkgModChunk) {
				throw new Error()
			}

			installRPKGMod()
		} catch {
			modChunkInputModalOpen = true
		}
	}}
/>

<TextInputModal
	bind:this={modChunkInputModal}
	showingModal={modChunkInputModalOpen}
	modalText="Mod chunk (if it advises you to name it chunk0patch3 or the file is named chunk0patchX, for example, then it's chunk0)"
	modalPlaceholder="chunk0"
	on:close={() => {
		rpkgModChunk = modChunkInputModal.value

		installRPKGMod()
	}}
/>

<Modal passiveModal open={rpkgModExtractionInProgress} modalHeading="Installing {rpkgModName}" preventCloseOnClickOutside>The mod is being installed - please wait.</Modal>

<Modal passiveModal open={frameworkModExtractionInProgress} modalHeading="Installing the mod" preventCloseOnClickOutside>The mod is being installed - please wait.</Modal>

<Modal alert bind:open={invalidFrameworkZipModalOpen} modalHeading="Invalid framework ZIP" primaryButtonText="OK" shouldSubmitOnEnter={false} on:submit={() => (invalidFrameworkZipModalOpen = false)}>
	<p>The framework ZIP file contains files in the root directory. Contact the mod author.</p>
</Modal>

<Modal alert bind:open={invalidModModalOpen} modalHeading="What" primaryButtonText="OK" shouldSubmitOnEnter={false} on:submit={() => (invalidModModalOpen = false)}>
	<p>This doesn't look like a mod? Make sure you select a mod ZIP, and that the mod is either a framework mod or RPKG mod.</p>
</Modal>

<Modal
	danger
	bind:open={frameworkModScriptsWarningOpen}
	modalHeading="Mod contains scripts"
	primaryButtonText="I'm sure"
	secondaryButtonText="Cancel"
	shouldSubmitOnEnter={false}
	on:click:button--secondary={() => (frameworkModScriptsWarningOpen = false)}
	on:click:button--primary={() => {
		window.fs.copySync("./staging", "../Mods")

		window.originalFs.writeFileSync(window.path.join("..", "Mods", window.fs.readdirSync("./staging")[0], "manifest.json:SMFExtractionTag"), "Extracted via SMF")

		window.fs.removeSync("./staging")

		window.location.reload()
	}}
>
	<p>
		This mod contains scripts; that means it is able to execute its own (external to the framework) code and effectively has complete control over your PC whenever you apply your mods. Scripts can
		do cool things and make a lot of mods possible, but they can also do bad things like installing malware on your computer. Make sure you trust whoever developed this mod, and wherever you
		downloaded it from. Are you sure you want to add this mod?
	</p>
</Modal>

<Modal
	alert
	bind:open={displayExtractedModsDialog}
	modalHeading="Incorrectly installed mod{extractedMods.length > 1 ? 's' : ''}"
	primaryButtonText="OK"
	shouldSubmitOnEnter={false}
	on:submit={() => (displayExtractedModsDialog = false)}
>
	<p>
		The mod{extractedMods.length > 1 ? "s" : ""}
		{extractedMods.slice(0, -1).length ? extractedMods.slice(0, -1).join(", ") + " and " + extractedMods[extractedMods.length - 1] : extractedMods[0]}
		{extractedMods.length > 1 ? "were" : "was"} installed by extracting the ZIP file directly to the Mods folder. That's not how you're meant to install mods; doing things this way could pose risks as
		it bypasses the framework's checks for mod validity and safety. Instead, use the Add a Mod button to add any mods you want. This message won't be shown again for {extractedMods.length > 1
			? "these mods"
			: "this mod"}.
		<br /><br />
		If you're seeing this after creating a new mod yourself, you should enable developer mode in the information page - it'll improve your experience and let you use the mod authoring tools in the Mod Manager.</p>
</Modal>

<Modal alert bind:open={uploadLogFailedModalOpen} modalHeading="Couldn't upload log" primaryButtonText="OK" shouldSubmitOnEnter={false} on:submit={() => (uploadLogFailedModalOpen = false)}>
	<p>Your log couldn't be uploaded. Make sure you're connected to the Internet.</p>
</Modal>

<Modal alert bind:open={uploadLogModalOpen} modalHeading="Log uploaded" primaryButtonText="OK" shouldSubmitOnEnter={false} on:submit={() => (uploadLogModalOpen = false)}>
	<p class="mb-2">Your deploy log has been anonymously uploaded to the Internet.</p>
	<CodeSnippet code={uploadedLogURL} />
	<br />
	<div class="mb-6" />
</Modal>

<style>
	:global(.bx--btn--ghost) {
		color: inherit;
		@apply bg-neutral-800;
	}

	:global(.bx--btn--ghost:hover, .bx--btn--ghost:active) {
		color: inherit;
	}

	/* Remove the weird spacing; it's created by the border, which can't be seen due to the dark background */
	:global(li) {
		border: inherit !important;
		transition: inherit !important;
	}

	:global(.over) {
		border-color: inherit !important;
	}

	:global(.bx--modal-close) {
		display: none;
	}

	:global(.bx--inline-notification__icon) {
		display: none;
	}

	:global(.bx--snippet.bx--snippet--single) {
		background-color: #262626;
	}
</style>
