<template>
  <t-modal
    name="paste-ascii-modal"
    header="Import from Clipboard"
    :clickToClose="false"
    :escToClose="true"
  >
    Title
    <t-input
      type="text"
      name="title"
      v-model="title"
      max="128"
    />

    <t-textarea v-model="pasteContent" name="paste-ascii" rows="10" />

    <template v-slot:footer>
      <div
        class="flex justify-between"
        @click="$modal.hide('paste-ascii-modal')"
      >
        <t-button type="button"> Cancel </t-button>
        <t-button type="button" @click="importPasteAscii()">Import Clipboard</t-button>
      </div>
    </template>
  </t-modal>
</template>

<script>
import LZString from "lz-string";

export default {
  name: "PasteAsciiModal",
  created() {
  },
  data: () => ({
    pasteContent: null,
    title: "clipboard.txt"
  }),
  computed: {
    showPasteModal() {
      return this.$store.getters.modalState.pasteModal;
    },
  },
  watch: {
    showPasteModal(val, old) {
        if (val !== old) {
            this.pasteModal();
        }

        // this.showPasteModal();
    },
  },
  methods: {
    pasteModal() {


      this.$modal.show("paste-ascii-modal");
    },
    create2DArray(rows) {
      const arr = [];

      for (let i = 0; i < rows; i++) {
        arr[i] = [];
      }

      return arr;
    },
    importPasteAscii() {
    const MIRC_MAX_COLOURS = this.$store.getters.mircColours.length;

      // The current state of the Colours
      let curBlock = {
        fg: null,
        bg: null,
        char: null,
      };

      let contents = this.pasteContent
      let filename = this.title

      // set asciiImport as the entire file contents as a string
      const asciiImport = contents
        .split("\u0003\u0003")
        .join("\u0003")
        .split("\u000F").join("")
        .split("\u0003\n").join("\n")
        .split("\u0002\u0003").join("\u0003");

      // This will end up in the asciibirdMeta
      const finalAscii = {
        width: false, // defined in: switch (curChar) case "\n":
        height: asciiImport.split("\n").length,
        title: filename,
        key: this.$store.getters.nextTabValue,
        blockWidth: 8 * this.$store.getters.blockSizeMultiplier,
        blockHeight: 13 * this.$store.getters.blockSizeMultiplier,
        blocks: this.create2DArray(asciiImport.split("\n").length),
        history: [],
        redo: [],
        x: 8 * 35, // the dragable ascii canvas x
        y: 13 * 2, // the dragable ascii canvas y
      };

      // Turn the entire ascii string into an array
      let asciiStringArray = asciiImport.split("");
      let linesArray = asciiImport.split("\n");

      // The proper X and Y value of the block inside the ASCII
      let asciiX = 0;
      let asciiY = 0;

      // used to determine colours
      let colourChar1 = null;
      let colourChar2 = null;
      let parsedColour = null;

      // This variable just counts the amount of colour and char codes to minus
      // to get the real width
      let widthOfColCodes = 0;

      // Better for colourful asciis
      let maxWidthLoop = 0;

      // Used before the loop, better for plain text
      let maxWidthFound = 0;

      for(let i = 0; i < linesArray.length; i++) {
        if (linesArray[i].length > maxWidthFound) {
          maxWidthFound = linesArray[i].length;
        }
      }

      while (asciiStringArray.length) {
        const curChar = asciiStringArray[0];

        // Defining a small finite state machine
        // to detect the colour code
        switch (curChar) {
          case "\n":
            // Reset the colours here on a new line
            curBlock = { 
              fg: null,
              bg: null,
              char: null,
            };

            if (linesArray[asciiY] && linesArray[asciiY].length > maxWidthLoop) {
              maxWidthLoop = linesArray[asciiY].length;
            }

            // the Y value of the ascii
            asciiY++;

            // Calculate widths mirc asciis vs plain text
            if (!finalAscii.width && widthOfColCodes > 0) {
              finalAscii.width =
                maxWidthLoop - widthOfColCodes; // minus \n for the proper width
            } 
            
            
            if (!finalAscii.width && widthOfColCodes === 0) {
              // Plain text
              finalAscii.width =
                maxWidthFound; // minus \n for the proper width
            }

            // Resets the X value
            asciiX = 0;

            asciiStringArray.shift();
            widthOfColCodes = 0;
            break;

          case "\u0003":
            // Remove the colour char
            asciiStringArray.shift();
            widthOfColCodes++;

            // Attempt to work out bg
            colourChar1 = `${asciiStringArray[0]}`;
            colourChar2 = `${asciiStringArray[1]}`;
            parsedColour = parseInt(`${colourChar1}${colourChar2}`);

            // Work out the 01, 02 double digit codes
            if (parseInt(colourChar1) === 0 && parseInt(colourChar2) >= 0) {
              asciiStringArray.shift();
            }

            if (isNaN(parsedColour)) {
              curBlock.bg = parseInt(colourChar1);
              widthOfColCodes += 1;
              asciiStringArray.shift();
            } else if (parsedColour <= MIRC_MAX_COLOURS && parsedColour >= 0) {
              curBlock.fg = parseInt(parsedColour);
              widthOfColCodes += parsedColour.toString().length;

              asciiStringArray = asciiStringArray.slice(
                parsedColour.toString().length,
                asciiStringArray.length
              );
            }

            // No background colour
            if (asciiStringArray[0] !== ",") {
              break;
            } else {
              // Remove , from array
              widthOfColCodes += 1;
              asciiStringArray.shift();
            }

            // Attempt to work out bg
            colourChar1 = `${asciiStringArray[0]}`;
            colourChar2 = `${asciiStringArray[1]}`;
            parsedColour = parseInt(`${colourChar1}${colourChar2}`);

            if (
              !isNaN(colourChar1) &&
              !isNaN(colourChar2) &&
              parseInt(colourChar2) > parseInt(colourChar1) &&
              !isNaN(parsedColour) &&
              parseInt(parsedColour) < 10
            ) {
              parsedColour = parseInt(colourChar2);
              widthOfColCodes += 1;
              asciiStringArray.shift();
            }

            if (
              parseInt(colourChar2) === parseInt(colourChar1) &&
              parseInt(parsedColour) < 10
            ) {
              parsedColour = parseInt(colourChar1);
              asciiStringArray.shift();
              asciiStringArray.shift();
              widthOfColCodes += 2;

              curBlock.bg = parseInt(colourChar1);

              break;
            }

            if (isNaN(parsedColour)) {
              curBlock.bg = parseInt(colourChar1);
              widthOfColCodes += 1;
              asciiStringArray.shift();
            } else if (parsedColour <= MIRC_MAX_COLOURS && parsedColour >= 0) {
              curBlock.bg = parseInt(parsedColour);
              widthOfColCodes += parsedColour.toString().length;

              asciiStringArray = asciiStringArray.slice(
                parsedColour.toString().length,
                asciiStringArray.length
              );

              break;
            }

            break;

          default:
            curBlock.char = curChar;
            asciiStringArray.shift();
            asciiX++;

            finalAscii.blocks[asciiY][asciiX - 1] = { ...curBlock };
            break;
        } // End Switch
      } // End loop charPos

      // Store the ASCII
      finalAscii.blocks = LZString.compressToUTF16(
        JSON.stringify(finalAscii.blocks)
      );
      finalAscii.history.push(finalAscii.blocks);
      this.$store.commit("newAsciibirdMeta", finalAscii);

      // To show the ASCII after importing we get the last key
      // from the asciiBirdMeta array
      const keys = this.$store.getters.asciibirdMeta
        .map((v, k) => k)
        .filter((i) => i !== undefined);

      // Set the current tab and pop the array for the last value
      this.currentTab = keys.pop();
      this.$store.commit("changeTab", this.currentTab);

      // Update the browsers title to the ASCII filename
      document.title = `asciibird - ${this.$store.getters.currentAscii.title}`;

        this.$modal.hide("paste-ascii-modal");
        
        this.pasteContent = null;
        this.title = 'clipboard.txt';
    },
  },
};
</script>
