<script>
  import { findMatchingItems, isDev, len, splitMax, trimPrefix } from "../util";
  import { focus } from "../actions";
  import ListBox from "./ListBox.svelte";
  import { extractShortcut } from "../keys";

  /** @typedef {[string, number]} CmdDef */

  /** @type {{
    onclose: () => void,
    executeCommand: (id: number) => void,
    switchToNoteSelector: () => void,
    commandsDef: CmdDef[],
}}*/
  let { onclose, executeCommand, switchToNoteSelector, commandsDef } = $props();

  /** @typedef {{
   key: number,
   name: string,
   nameLC: string,
   shortcut: string,
   ref: HTMLElement,
  }} Item 
   */

  function verifyCommandsAreUnique() {
    let m = new Map();
    for (let i = 0; i < len(commandsDef); i++) {
      let s = commandsDef[i][0];
      let id = commandsDef[i][1];
      if (m.has(id)) {
        console.log(`Duplicate items with id ${id}: '${s}' and '${m.get(id)}'`);
      }
      m.set(id, s);
    }
  }

  /**
   * @returns {Item[]}
   */
  function buildCommands() {
    if (isDev()) {
      verifyCommandsAreUnique();
    }

    // console.log("rebuildCommands:", commands);
    /** @type {Item[]} */
    let res = Array(len(commandsDef));
    for (let i = 0; i < len(commandsDef); i++) {
      let s = commandsDef[i][0];
      let id = commandsDef[i][1];
      let parts = splitMax(s, "\t", 2);
      let name = parts[0];
      let shortcut = null;
      if (len(parts) > 1) {
        shortcut = extractShortcut(parts[1]);
      }
      // console.log(`i: ${i}, name: ${name} id: ${id}`);
      let item = {
        key: id,
        name: name,
        nameLC: name.toLowerCase(),
        shortcut: shortcut,
        ref: null,
      };
      res[i] = item;
    }
    // -1 if a < b
    // 0 if a = b
    // 1 if a > b
    res.sort((a, b) => {
      return a.name.localeCompare(b.name);
    });
    return res;
  }
  let items = $state(buildCommands());
  let cmdCountMsg = `${len(items)} commands`;
  let filter = $state(">");

  $effect(() => {
    if (filter === "") {
      switchToNoteSelector();
    }
  });

  let itemsFiltered = $derived.by(() => {
    let lc = filter.toLowerCase();
    lc = trimPrefix(lc, ">");
    return findMatchingItems(items, lc, "nameLC");
  });

  /**
   * @param {Item} item
   */
  function emitExecuteCommand(item) {
    console.log("emitExecuteCommand", $state.snapshot(item));
    executeCommand(item.key);
  }

  /**
   * @param {KeyboardEvent} ev
   */
  function onkeydown(ev) {
    let key = ev.key;
    if (key === "Escape") {
      if (filter !== ">") {
        filter = ">";
      } else {
        onclose();
      }
      ev.preventDefault();
      ev.stopPropagation();
      return;
    }

    let allowLeftRight = filter === "";
    listbox.onkeydown(ev, allowLeftRight);
  }

  let listbox;
</script>

<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
<form
  {onkeydown}
  tabindex="-1"
  class="fixed inset-0 overflow-hidden flex flex-col z-20 w-full h-full p-2 text-sm bg-white dark:bg-gray-900 dark:text-gray-300"
>
  <div>
    <input
      type="text"
      use:focus
      bind:value={filter}
      class="py-1 px-2 bg-white w-full mb-2 rounded-xs relative"
    />
    <div class="absolute right-[1rem] top-[0.75rem] italic text-gray-400">
      {cmdCountMsg}
    </div>
  </div>
  <ListBox
    bind:this={listbox}
    items={itemsFiltered}
    onclick={(item) => emitExecuteCommand(item)}
  >
    {#snippet renderItem(item)}
      <div class="truncate">
        {item.name}
      </div>
      <div class="grow"></div>
      <div class="mr-2">{item.shortcut}</div>
    {/snippet}
  </ListBox>
</form>
