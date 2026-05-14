<script setup lang="ts">
  import { ref, type Ref } from 'vue';
  import axios from 'axios';
  import Swal from 'sweetalert2';


  const placeholder_textarea: Ref<string> = ref('Pokemon: 22\n4 Dreepy TWM 128\n4 Drakloak TWM 129\n3 Dragapult ex TWM 130\n...')
  const cards_with_errors: Ref<string[]> = ref([]);
  const input_data: Ref<string> = ref('');
  const input_format: Ref<string> = ref('');
  const output_data: Ref<string> = ref('');
  const output_format: Ref<string> = ref('');
  const language: Ref<string> = ref('en');
  const handle_duplicates: Ref<string> = ref('none');
  const lowest_price_cardtrader: Ref<boolean> = ref(false);

  const loading: Ref<boolean> = ref(false);

  type Card = {
      number: number;
      card: string;
      expansion: string;
      expansion_number: string;
  };

  type Expansion = {
    symbol: string;
    api_id: number;
    name: string;
  };

  type TcgDexCard = {
    abilities: {name: string, [key: string]: any}[],
    attacks: {name: string, [key: string]: any}[],
    name: string,
    image: string,
    [key: string]: any,
    category: string,
    number_cards_requested: number
  }

  function get_cards_from_limitless_format(): [cards_requested_object: Card[], expansions_requested: string[]] {
    let expansions_requested: string[] = [];
    const cards_requested = input_data.value.split('\n').filter(card => card !== '' && !/(Pok[eé]mon|Trainer|Energy): (\d)+/.test(card))
    
    let cards_requested_object: Card[] = []
    cards_requested.forEach((card) => {
      const match = card.match(/^(\d+)\s(.*)\s([A-Za-z0-9]+)\s(\d+)$/)
      if (match == undefined) {
        cards_with_errors.value.push(card)
      } else{
        expansions_requested.push(match[3]);
        cards_requested_object.push({"number": Number(match[1]), "card": match[2], "expansion": match[3], "expansion_number": match[4]})
      }
    })

    if (handle_duplicates.value != 'none') {
      const grouped = Object.values(
        cards_requested_object.reduce<Record<string, Card>>((acc, { number, ...rest }) => {
          const key = JSON.stringify(rest)
          
          if (handle_duplicates.value == 'max_4') {
              if (!acc[key]) {
                acc[key] = { ...rest, number: 0 }
              }

              acc[key].number += number;

              if (acc[key].number > 4) {
                acc[key].number = 4
              }
          }
          else if (handle_duplicates.value == "max_any_reference") {
            if (!acc[key]) {
              acc[key] = { ...rest, number }
            } else {
              acc[key].number = Math.max(acc[key].number, number)
            }
          }
          

          return acc
        }, {})
      )

      cards_requested_object = grouped;
    }

    expansions_requested = [...new Set(expansions_requested)];

    return [cards_requested_object, expansions_requested]
  }

  async function get_expansions_info(expansions_requested: string[]): Promise<Expansion[]> {
    let expansions_info: Expansion[] = [];

    for (const expansion of expansions_requested) {
      await axios.get(
        `https://api.tcgdex.net/v2/${language.value}/sets?abbreviation.official=eq:${expansion}`
      ).then(function(response) {
        if (response.data.length > 0) {
          expansions_info.push({
            'symbol': expansion,
            'api_id': response.data[0].id,
            'name': response.data[0].name
          })
        }
      })

    }
    return expansions_info
  }

  async function get_cards(cards_requested_object: Card[], expansions_info: Expansion[]): Promise<TcgDexCard[]> {
    let cards_info: TcgDexCard[] = [];

    for (const card of cards_requested_object) {
      let expansion_items: Expansion[] = expansions_info.filter(expansion => expansion.symbol == card.expansion)
      if (expansion_items.length != 1) {
        cards_with_errors.value.push(`${card.number} ${card.card} ${card.expansion} ${card.expansion_number}`)
      } else {
        const expansion_id: number = expansion_items[0].api_id

        await axios.get(
          `https://api.tcgdex.net/v2/${language.value}/sets/${expansion_id}/${card.expansion_number}`
        ).then(function(response) {
          if (response.data) {
            const tcgdex_card = response.data;
            tcgdex_card.number_cards_requested = card.number;
            cards_info.push(tcgdex_card);
          } else {
            cards_with_errors.value.push(`${card.number} ${card.card} ${card.expansion} ${card.expansion_number}`)
          }
        })
      }
      
    }

    return cards_info;
  }


  async function get_cheapeast_version_of_cards(cards: TcgDexCard[]): Promise<TcgDexCard[]> {
    let cheapeast_versions: TcgDexCard[] = [];

    for (const card of cards) {
      let attacks_string = ""
      if (["Pokemon", "Pokémon"].includes(card.category)) {
        card.attacks.forEach((attack, idx) => {
          attacks_string += `&attacks.${idx}.name=eq:${attack.name}`
        })
      } 
      
      let best_version: TcgDexCard | null = null;

      await axios.get(
      `https://api.tcgdex.net/v2/${language.value}/cards?name=eq:${card.name}${attacks_string}`
      ).then(async function(response) {
        if (response.data) {
          const all_versions_available = response.data;
          for (const version of all_versions_available) {
            const card_expansion_number = version.localId;
            const expansion_id = version.id.split('-' + card_expansion_number)[0];
            await axios.get(
              `https://api.tcgdex.net/v2/${language.value}/sets/${expansion_id}/${card_expansion_number}`
            ).then(function(response) {
              if (response.data) {
                const tcgdex_card = response.data;
                if (
                  best_version == null || (tcgdex_card.pricing?.cardmarket?.avg !== undefined && best_version.pricing?.cardmarket?.avg === undefined) ||
                  tcgdex_card.pricing?.cardmarket?.avg < best_version.pricing?.cardmarket?.avg) {
                  best_version = tcgdex_card
                }
              } 
            })
          }
          if (best_version != null) {
            best_version.number_cards_requested = card.number_cards_requested;
            cheapeast_versions.push(best_version)
          } else {
            cards_with_errors.value.push(`${card.number} ${card.card} ${card.expansion} ${card.expansion_number}`)
          }
        }
      })
    }
    
    return cheapeast_versions;
  }


  async function convert_cards_to_cardmarket_format(expansions_requested: string[], cards_requested_object: Card[]) {
    const expansions_info: Expansion[] = await get_expansions_info(expansions_requested);

    const cards_info: TcgDexCard[] = await get_cards(cards_requested_object, expansions_info);

    for (const card of cards_info) {
      if (["Pokemon", "Pokémon"].includes(card.category)) {
        const abilities_and_attacks = [...card.abilities || [], ...card.attacks];
        output_data.value += `${card.number_cards_requested} ${card.name} [${abilities_and_attacks.map(element => element.name).join(' | ')}]\n`
      } else {
        const boss_order_names = {
          "en": "Boss's Orders",
          "es": "Órdenes de Jefes",
          "de": "Befehl vom Boss",
          "it": "Ordini del Capo",
          "fr": "Ordres du Boss",
          "pt": "Ordem da Chefia"
        }
        for (const [language_boss_order, card_name] of Object.entries(boss_order_names)) {
          if (card.name == card_name && language_boss_order == language.value) {
            card.name += " - Giovanni";
          }
        }
        output_data.value += `${card.number_cards_requested} ${card.name}\n`
      }
    }

  }

  async function convert_cards_to_cardtrader_format(expansions_requested: string[], cards_requested_object: Card[], lowest_price: boolean) {
    if (lowest_price) {
      const energy_cards: Card[] = cards_requested_object.filter(card => [
        "Basic Grass Energy", "Basic Fire Energy", "Basic Water Energy", "Basic Lightning Energy", "Basic Colorless Energy",
        "Basic Psychic Energy", "Basic Fighting Energy", "Basic Darkness Energy", "Basic Metal Energy", "Basic Fairy Energy",
        "Grass Energy", "Fire Energy", "Water Energy", "Lightning Energy", "Colorless Energy",
        "Psychic Energy", "Fighting Energy", "Darkness Energy", "Metal Energy", "Fairy Energy",
      ].includes(card.card));
      
      const cards_requested = cards_requested_object.filter(card => !energy_cards.includes(card));

      const expansions_info: Expansion[] = await get_expansions_info(expansions_requested);

      const cards_info: TcgDexCard[] = await get_cards(cards_requested, expansions_info);

      const cheapest_cards: TcgDexCard[] = await get_cheapeast_version_of_cards(cards_info);
      

      const new_expansions_ids: string[] = [...new Set(cheapest_cards.map(card => card.set.id))];

      const new_expansions: Expansion[] = [];
      for (const expansion of new_expansions_ids) {
        await axios.get(
          `https://api.tcgdex.net/v2/${language.value}/sets/${expansion}`
        ).then(function(response) {
          if (response.data) {
            new_expansions.push({
              "api_id": response.data.id,
              "name": response.data.name,
              "symbol": response.data.abbreviation.official
            })
          }
        })
      }
      
      for (const card of cheapest_cards) {
        let expansion_items: Expansion[] = new_expansions.filter(expansion => expansion.api_id == card.set.id)
        if (expansion_items.length != 1) {
          cards_with_errors.value.push(`${card.number_cards_requested} ${card.name} ${card.expansion} ${card.expansion_number}`)
        } else {
          output_data.value += `${card.number_cards_requested} ${card.name} (${expansion_items[0].symbol})\n`
        }
      }
    } else {
      for (const card of cards_requested_object) {
        output_data.value += `${card.number} ${card.card} (${card.expansion})\n`
    }
    }
    

  }


  async function generate_cards(){
    loading.value = true;
    cards_with_errors.value = [];
    output_data.value = "";

    let cards_requested_object: Card[] = [];
    let expansions_requested: string[] = [];

    if (input_format.value == 'limitless') {
      [cards_requested_object, expansions_requested] = get_cards_from_limitless_format();
    } else {
      Swal.fire({
        icon: 'error',
        title: 'Unexpected error',
        text: 'Unexpected input format selected: ' + input_format.value,
      })
      loading.value = false;
      return
    }

    if (output_format.value == 'cardmarket') {
      await convert_cards_to_cardmarket_format(expansions_requested, cards_requested_object);
    }
    else if (output_format.value == 'cardtrader') {
      await convert_cards_to_cardtrader_format(expansions_requested, cards_requested_object, lowest_price_cardtrader.value);
    }
    else {
      Swal.fire({
        icon: 'error',
        title: 'Unexpected error',
        text: 'Unexpected output format selected: ' + output_format.value,
      })
      loading.value = false;
      return
    }

    if (cards_with_errors.value.length > 0) {
      Swal.fire({
        icon: 'warning',
        title: 'Cards with errors',
        html: cards_with_errors.value.join('<br/>'),
      })
    }
    

    loading.value = false;

  }



</script>

<template>
  <main>    
    <section id="parameters">
      <div>
        <p>This web application is a tool to convert your Pokémon TCG list from one format to another.</p>
        <p>Select input and output formats, your language, how to handle duplicates, include your deck, and click on "Convert" button.</p>
        <p><em>Note about how to handle duplicates: </em>If <em>"Max 4 of each card"</em> is selected, TCGListConverter will add the maximum of 4 of each card to your deck. If <em>"Max as cards as indicated in any line"</em> is selected, TCGListConverter will add only the maximum of a card indicated in any reference to that card (if a card is indicated to be included 2 and 3 times in two lines, only 3 of this card will be included).</p>
      </div>
      <div id="input_options">
        <img width="150" src="../assets/limitless.png" alt="limitless" @click="input_format = 'limitless'" :class="input_format == 'limitless' ? 'selected_img' : 'unselected_img'"/>
      </div>
      <div id="output_options">
        <img width="150" src="../assets/cardmarket.png" alt="cardmarket" @click="output_format = 'cardmarket'" :class="output_format == 'cardmarket' ? 'selected_img' : 'unselected_img'"/>
        <img width="150" src="../assets/cardtrader.png" alt="cardtrader" @click="output_format = 'cardtrader'" :class="output_format == 'cardtrader' ? 'selected_img' : 'unselected_img'"/>
      </div>
      <select name="language" id="language" v-model="language">
        <option value="en">English</option>
        <option value="es">Spanish</option>
        <option value="fr">French</option>
        <option value="de">German</option>
        <option value="it">Italian</option>
        <option value="pt">Portuguese</option>
      </select>
      <select name="handle_duplicates" id="handle_duplicates" v-model="handle_duplicates">
        <option value="none">No actions</option>
        <option value="max_4">Max 4 of each card</option>
        <option value="max_any_reference">Max as cards as indicated in any line</option>
      </select>
      <label v-show="output_format=='cardtrader'">
        <input type="checkbox" id="lowest_price_cardtrader" v-model="lowest_price_cardtrader" /> Search for set with lowest price (basic energies will be removed)
      </label>
    </section>  
    
    <div id="input_output">
      <section id="input_section">
        <textarea id="input_textarea" :placeholder="placeholder_textarea" rows="30" cols="40" v-model="input_data"></textarea>
      </section>
      <img id="convert_img" src="../assets/convert.png" @click="generate_cards()" width="100" :class="{spinning: loading}" alt="Convert"/>
      <section id="output_section">
        <textarea readonly="true" rows="30" cols="40" v-model="output_data" placeholder="This text area will include your cards in the desired format"></textarea>
      </section>
    </div>
  </main>
  <section id="additional_info">
    <p>This webpage uses <a href="https://tcgdex.dev" target="_blank">TCGdex API</a> to fetch relevant information to convert your deck. This website is not affiliated nor related to the TCGdex project.</p>
    <p><em>Note 1:</em> Boss's Orders card has multiple versions and TCGdex API does not provide us information about which version is requested. Therefore, this webpage returns always "Giovanni" version of Boss's Orders.</p>
    <p><em>Note 2:</em> We have found that Switch card is not properly imported with Cardtrader import function. Therefore, you will need to add it manually.</p>
  </section>
</template>

<style>
  body {
    background-color: #0f172a;
    min-height: 100dvh;
    height: 100dvh;
    margin: 0;
  }

  #app {
    min-height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }

</style>

<style scoped>
  p, label {
    color: #e5e7eb;
  }

  label {
    background-color: transparent;
    color: #e5e7eb;

    border: 1px solid #374151;
    border-radius: 8px;
    padding: 8px 12px;
  }

  #toggle-theme {
    position: fixed;
    top: 1dvh;
    right: 1dvw;
  }

  select {
    background-color: transparent;
    color: #e5e7eb;

    border: 1px solid #374151;
    border-radius: 8px;

    padding: 8px 12px;

    outline: none;
    appearance: none;
  }

  select:hover {
    border-color: #4b5563;
  }

  select:focus {
    border-color: #60a5fa;
    box-shadow: 0 0 0 2px rgba(96, 165, 250, 0.3);
  }

  main {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-around;
    gap: 5rem;
    padding: 3rem 7dvw;
    margin: auto;
  }

  #input_output {
    display: flex;
    flex-direction: row;
    flex: 7;
    align-items: center;
    justify-content: space-around;
    gap: 0.5rem;
  }

  #parameters {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
    align-items: center;
    justify-content: center;
    flex: 3;
    max-width: 90dvw;
  }

  #convert_img {
    flex: 1;
  }

  #input_options, #output_options {
    border: 1px solid #374151;
    border-radius: 10px;
    padding: 1rem;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-around;
    gap: 2rem;
  }

  #input_section, #output_section {
    display: flex;
    flex-direction: column;
  }

  .unselected_img {
    opacity: 0.3;
  }

  #input_options > img, #output_options > img, #toggle-theme > img, #convert_img {
    cursor: pointer;
  }

  #input_section, #output_section {
    flex: 4;
  }

  #additional_info {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: auto;
    margin-bottom: 1rem;
    max-width: 90dvw;
  }

  #additional_info > p {
    margin: 0.5rem;
  }

  a {
  color: #60a5fa;
  text-decoration: none;
}

  a:visited {
    color: #a78bfa;
  }

  a:hover {
    color: #93c5fd;
    text-decoration: underline;
  }

  a:active {
    color: #bfdbfe;
  }
  
  @keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
  }

  .spinning {
    animation: spin 2s linear infinite;
  }

  @media (max-width: 1280px) {
    main {
      flex-direction: column;
    }
  }

  @media (max-width: 900px) {
    #input_output {
      flex-direction: column;
      margin: auto;
    }

    textarea {
      width: 80dvw;
    }

    main {
      margin: 0;
    }

    #output_options {
      flex-direction: column;
    }
  }

  
</style>