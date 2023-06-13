<script lang="ts">
  import { applyProfile, getDefaultProfileKey, getProfile, getProfileSelect } from './Profiles.svelte'
  import { getChatDefaults, getChatSettingList, getChatSettingObjectByKey, getExcludeFromProfile } from './Settings.svelte'
  import {
    saveChatStore,
    apiKeyStorage,
    chatsStorage,
    globalStorage,
    saveCustomProfile,
    deleteCustomProfile,
    setGlobalSettingValueByKey,
    resetChatSettings,
    checkStateChange,
    addChat
  } from './Storage.svelte'
  import type { Chat, ChatSetting, ResponseModels, SettingSelect, SelectOption, ChatSettings } from './Types.svelte'
  import { errorNotice, sizeTextElements } from './Util.svelte'
  import Fa from 'svelte-fa/src/fa.svelte'
  import {
    faTrash,
    faClone,
    faEllipsis,
    faFloppyDisk,
    faThumbtack,
    faDownload,
    faUpload,
    faSquarePlus,

    faRotateLeft

  } from '@fortawesome/free-solid-svg-icons/index'
  import { exportProfileAsJSON } from './Export.svelte'
  import { onMount, afterUpdate } from 'svelte'
  import ChatSettingField from './ChatSettingField.svelte'
  import { getModelMaxTokens } from './Stats.svelte'
  import { replace } from 'svelte-spa-router'
  import { openModal } from 'svelte-modals'
  import PromptConfirm from './PromptConfirm.svelte'
  import { getApiBase, getEndpointModels } from './ApiUtil.svelte'
  import { supportedModelKeys } from './Models.svelte'

  export let chatId:number
  export const show = () => { showSettings() }
  
  let showSettingsModal = 0
  let showProfileMenu:boolean = false
  let profileFileInput
  let defaultProfile = getDefaultProfileKey()
  let isDefault = false

  const settingsList = getChatSettingList()
  const modelSetting = getChatSettingObjectByKey('model') as ChatSetting & SettingSelect
  const chatDefaults = getChatDefaults()
  const excludeFromProfile = getExcludeFromProfile()

  $: chat = $chatsStorage.find((chat) => chat.id === chatId) as Chat
  $: chatSettings = chat.settings
  $: globalStore = $globalStorage

  let originalProfile:string
  let originalSettings:ChatSettings

  onMount(async () => {
    originalProfile = chatSettings && chatSettings.profile
    originalSettings = chatSettings && JSON.parse(JSON.stringify(chatSettings))
  })

  afterUpdate(() => {
    if (!originalProfile) {
      originalProfile = chatSettings && chatSettings.profile
      originalSettings = chatSettings && JSON.parse(JSON.stringify(chatSettings))
    }
    sizeTextElements()
  })
  
  const closeSettings = () => {
    originalProfile = ''
    originalSettings = {} as ChatSettings
    showProfileMenu = false
    $checkStateChange++
    showSettingsModal = 0
  }

  const clearSettings = () => {
    openModal(PromptConfirm, {
      title: 'Reset Changes',
      message: 'Are you sure you want to reset all changes you\'ve made to this profile?',
      class: 'is-warning',
      onConfirm: () => {
        resetChatSettings(chatId)
        refreshSettings()
      }
    })
  }

  const refreshSettings = async () => {
    showSettingsModal && showSettings()
  }
  
  const cloneProfile = () => {
    showProfileMenu = false
    const clone = JSON.parse(JSON.stringify(chat.settings))
    const name = chat.settings.profileName
    clone.profileName = newNameForProfile(name || '')
    clone.profile = null
    try {
      saveCustomProfile(clone)
      chat.settings.profile = clone.profile
      chat.settings.profileName = clone.profileName
      applyProfile(chatId, clone.profile)
      refreshSettings()
    } catch (e) {
      errorNotice('Error cloning profile:', e)
    }
  }

  const promptDeleteProfile = () => {
    openModal(PromptConfirm, {
      title: 'Delete Profile',
      message: 'Are you sure you want to delete this profile?',
      class: 'is-warning',
      onConfirm: () => {
        deleteProfile()
      }
    })
  }

  const deleteProfile = () => {
    showProfileMenu = false
    try {
      deleteCustomProfile(chatId, chat.settings.profile)
      chat.settings.profile = globalStore.defaultProfile || ''
      saveChatStore()
      setGlobalSettingValueByKey('lastProfile', chat.settings.profile)
      applyProfile(chatId, chat.settings.profile)
      refreshSettings()
    } catch (e) {
      console.error(e)
      errorNotice('Error deleting profile:', e)
    }
  }

  const pinDefaultProfile = () => {
    showProfileMenu = false
    setGlobalSettingValueByKey('defaultProfile', chat.settings.profile)
    refreshSettings()
  }

  const importProfileFromFile = (e) => {
    const image = e.target.files[0]
    const reader = new FileReader()
    reader.readAsText(image)
    reader.onload = e => {
      const json = (e.target || {}).result as string
      try {
        const profile = JSON.parse(json)
        profile.profileName = newNameForProfile(profile.profileName || '')
        profile.profile = null
        saveCustomProfile(profile)
        refreshSettings()
      } catch (e) {
        errorNotice('Unable to import profile:', e)
      }
    }
  }

  const updateProfileSelectOptions = () => {
    const profileSelect = getChatSettingObjectByKey('profile') as ChatSetting & SettingSelect
    profileSelect.options = getProfileSelect()
    chatDefaults.profile = getDefaultProfileKey()
    chatDefaults.max_tokens = getModelMaxTokens(chatSettings.model)
    // const defaultProfile = globalStore.defaultProfile || profileSelect.options[0].value
    defaultProfile = getDefaultProfileKey()
    isDefault = defaultProfile === chatSettings.profile
  }
  
  const showSettings = async () => {
    setDirty()
    // Show settings modal
    showSettingsModal++

    // Get profile options
    updateProfileSelectOptions()

    // Refresh settings modal
    showSettingsModal++
  
    // Load available models from OpenAI
    const allModels = (await (
      await fetch(getApiBase() + getEndpointModels(), {
        method: 'GET',
        headers: {
          Authorization: `Bearer ${$apiKeyStorage}`,
          'Content-Type': 'application/json'
        }
      })
    ).json()) as ResponseModels
    const filteredModels = supportedModelKeys.filter((model) => allModels.data.find((m) => m.id === model))

    const modelOptions:SelectOption[] = filteredModels.reduce((a, m) => {
      const o:SelectOption = {
        value: m,
        text: m
      }
      a.push(o)
      return a
    }, [] as SelectOption[])

    // Update the models in the settings
    if (modelSetting) {
      modelSetting.options = modelOptions
    }
    // Refresh settings modal
    showSettingsModal++

    setTimeout(() => sizeTextElements(), 0)
  }

  const saveProfile = () => {
    showProfileMenu = false
    try {
      saveCustomProfile(chat.settings)
      refreshSettings()
    } catch (e) {
      errorNotice('Error saving profile:', e)
    }
  }

  const newNameForProfile = (name:string):string => {
    const profiles = getProfileSelect()
    const nameMap = profiles.reduce((a, p) => { a[p.text] = p; return a }, {})
    if (!nameMap[name]) return name
    let i:number = 1
    let cname = name + `-${i}`
    while (nameMap[cname]) {
      i++
      cname = name + `-${i}`
    }
    return cname
  }

  const startNewChat = () => {
    const differentProfile = originalSettings.profile !== chatSettings.profile
    // start new
    const newChatId = addChat(chatSettings)
    // restore original
    if (differentProfile) {
      chat.settings = originalSettings
      saveChatStore()
    }
    // go to new chat
    replace(`/chat/${newChatId}`)
  }

  const deepEqual = (x:any, y:any) => {
    const ok = Object.keys; const tx = typeof x; const ty = typeof y
    return x && y && tx === 'object' && tx === ty
      ? (
          ok(x).every(key => excludeFromProfile[key] || deepEqual(x[key], y[key]))
        )
      : (x === y || ((x === undefined || x === null || x === false) && (y === undefined || y === null || y === false)))
  }

  const setDirty = (e:CustomEvent|undefined = undefined) => {
    if (e) {
      const setting = e.detail as ChatSetting
      const key = setting.key
      if (key === 'profile') return
    }
    const profile = getProfile(chatSettings.profile)
    chatSettings.isDirty = !deepEqual(profile, chatSettings)
  }

</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="modal chat-settings" class:is-active={showSettingsModal} on:modal-esc={closeSettings}>
  <div class="modal-background" on:click={closeSettings} />
  <div class="modal-card wide" on:click={() => { showProfileMenu = false }}>
    <header class="modal-card-head">
      <p class="modal-card-title">Chat Settings</p>
      <button class="delete" aria-label="close" on:click={closeSettings}></button>
    </header>
    <section class="modal-card-body">
      {#key showSettingsModal}
      {#each settingsList as setting}
        <ChatSettingField on:refresh={refreshSettings} on:change={setDirty} chat={chat} chatDefaults={chatDefaults} chatSettings={chatSettings} setting={setting} originalProfile={originalProfile} />
      {/each}
      {/key}
    </section>

    <footer class="modal-card-foot">
      <div class="level is-mobile">
        <div class="level-left">
          <!-- <button class="button is-info" on:click={closeSettings}>Close</button> -->
          <button class="button" title="Save changes to this profile." class:is-disabled={!chatSettings.isDirty} on:click={saveProfile}>Save</button>    
          <button class="button is-warning" title="Throw away changes to this profile." class:is-disabled={!chatSettings.isDirty} on:click={clearSettings}>Reset</button>
          <button class="button" title="Start new chat with this profile." on:click={startNewChat}>New Chat</button>
        </div>
        <div class="level-right">
          <div class="dropdown is-right is-up" class:is-active={showProfileMenu}>
            <div class="dropdown-trigger">
              <button class="button" aria-haspopup="true" aria-controls="dropdown-menu3" on:click|preventDefault|stopPropagation={() => { showProfileMenu = !showProfileMenu }}>
                <span class="icon"><Fa icon={faEllipsis}/></span>
              </button>
            </div>
            <div class="dropdown-menu" id="dropdown-menu3" role="menu">
              <div class="dropdown-content">
                <a href={'#'} class="dropdown-item" class:is-disabled={!chatSettings.isDirty} on:click|preventDefault={saveProfile}>
                  <span class="menu-icon"><Fa icon={faFloppyDisk}/></span> Save Changes
                </a>
                <a href={'#'} class="dropdown-item" class:is-disabled={!chatSettings.isDirty} on:click|preventDefault={clearSettings}>
                  <span class="menu-icon"><Fa icon={faRotateLeft}/></span> Reset Changes
                </a>
                <a href={'#'} class="dropdown-item" on:click|preventDefault={cloneProfile}>
                  <span class="menu-icon"><Fa icon={faClone}/></span> Clone Profile
                </a>
                <hr class="dropdown-divider">
                <a href={'#'} class="dropdown-item" class:is-disabled={isDefault} on:click|preventDefault={pinDefaultProfile}>
                  <span class="menu-icon"><Fa icon={faThumbtack}/></span> Set as Default Profile
                </a>
                <a href={'#'} class="dropdown-item" on:click|preventDefault={startNewChat}>
                  <span class="menu-icon"><Fa icon={faSquarePlus}/></span> Start New Chat Using Profile
                </a>
                <hr class="dropdown-divider">
                <a href={'#'} 
                  class="dropdown-item"
                  on:click|preventDefault={() => { showProfileMenu = false; exportProfileAsJSON(chatId) }}
                >
                  <span class="menu-icon"><Fa icon={faDownload}/></span> Backup Profile JSON
                </a>
                <a href={'#'} class="dropdown-item" on:click|preventDefault={() => { showProfileMenu = false; profileFileInput.click() }}>
                  <span class="menu-icon"><Fa icon={faUpload}/></span> Restore Profile JSON
                </a>
                <hr class="dropdown-divider">
                <a href={'#'} class="dropdown-item" on:click|preventDefault={promptDeleteProfile}>
                  <span class="menu-icon"><Fa icon={faTrash}/></span> Delete Profile
                </a>
              </div>
            </div>
          </div>
        </div>
      </div>  
    </footer>
  </div>
</div>

<input style="display:none" type="file" accept=".json" on:change={(e) => importProfileFromFile(e)} bind:this={profileFileInput} >