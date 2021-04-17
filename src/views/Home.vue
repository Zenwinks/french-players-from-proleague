<template>
  <div class="home">
    <div class="leagues">
      <div class="head">
        <h3>Choisissez une ou plusieurs ligues</h3>
        <button class="btn btn-secondary btn-sm" type="button" @click="changeSelection()">Tout
          sélectionner/Déséletionner
        </button>
      </div>
      <div class="leagues-block">
        <div v-for="league in leagues" :key="league.id" class="form-group form-check">
          <input :id="'check'+league.id" v-model="league.checked" class="form-check-input" type="checkbox">
          <label :for="'check'+league.id" class="form-check-label">{{ league.name }}</label>
        </div>
      </div>
      <button class="btn btn-primary btn-lg btn-block search-btn" type="button" @click="launchSearch()">
        <img
            alt="Logo EDF"
            src="../../public/favicon.png">
        Rechercher
      </button>
    </div>
    <div class="players">
      <h3>{{ frenchPlayers.length }} joueurs français trouvé(s)</h3>
      <div v-if="loading" class="loading">
        <span>Chargement en cours...</span>
      </div>
      <div v-else-if="frenchPlayers.length > 0" class="players-block">
        <div v-for="player in frenchPlayers" :key="player.id" class="player">
          <div class="profilepic">
            <img :src="player.logo" alt="Team logo">
          </div>
          <div class="data">
            <input :value="player.id" type="hidden">
            <div class="name">
              <a :href="'https://proleague.de/profil.php?id='+player.id" target="_blank">{{ player.name }}</a>
            </div>
            <div class="team"><span>Équipe : </span><span>{{ player.team }}</span></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios"

export default {
  name: 'Home',
  data() {
    return {
      leagues: [
        {id: "1", name: "Allemagne 1", checked: true},
        {id: "2", name: "Allemagne 2", checked: true},
        {id: "3", name: "Allemagne 3", checked: true},
        {id: "int1", name: "Inter 1", checked: true},
        {id: "int2", name: "Inter 2", checked: true},
        {id: "int3", name: "Inter 3", checked: true},
        {id: "tr1", name: "Turquie 1", checked: true},
        {id: "pol1", name: "Pologne 1", checked: true},
        {id: "rom1", name: "Roumanie 1", checked: true},
        {id: "ita1", name: "Italie 1", checked: true},
        {id: "ita2", name: "Italie 2", checked: true},
        {id: "fra1", name: "France 1", checked: true},
        {id: "spa1", name: "Espagne 1", checked: true},
        {id: "uk1", name: "Royaume-Uni/Irlande 1", checked: true}
      ],
      frenchPlayers: [],
      loading: false,
      lastTeam: false
    }
  },
  methods: {
    changeSelection() {
      this.leagues.map(league => {
        league.checked = !league.checked
      })
    },
    launchSearch() {
      this.frenchPlayers = []
      this.lastTeam = false
      this.loading = true
      console.log("Récupération de tous les joueurs français...")
      this.leagues.forEach(league => {
        if (league.checked) {
          this.getLeagues(league.id)
        }
      })
    },
    getLeagues(id) {
      return new Promise(() => {
        axios.get('https://proleague.de/league.php?league=' + id).then(response => {
          this.getTeamsByLeague(response.data, 1)
        })
      })
    },
    getTeamsByLeague(league, index) {
      console.log("-----------------------------")
      var newLeague = league.split('id="leaguetable">')[1].split('</table>')[0]
      var indexTeam = newLeague.indexOf("<a href=\"team.php", (index + 1))
      var firstStep = newLeague.substring(indexTeam, (indexTeam + 200))
      var id = firstStep.substring(firstStep.indexOf("?id="), firstStep.indexOf("#team\">")).substring(4)
      var name = firstStep.substring(firstStep.indexOf("#team\">"), firstStep.indexOf("</a>")).substring(7)
      if (name.includes("1iQ Gang")) {
        name = name.substring(0, 8)
      } else if (name.includes("Venga Venga FC")) {
        name = name.substring(0, 14)
      }
      console.log('ID nouvelle équipe : ' + id)
      console.log('Nom nouvelle équipe : ' + name)
      this.getTeamById(id, name)
      var newIndex = newLeague.indexOf("<a href=\"team.php", (indexTeam + 1))
      if (newIndex !== -1 && newIndex !== indexTeam) {
        this.getTeamsByLeague(league, indexTeam)
      } else {
        this.lastTeam = true
      }
    },
    getTeamById(id, name) {
      axios.get('https://proleague.de/team.php?id=' + id).then(response => {
        this.getPlayersByTeam(response.data, 1, id, name)
      })
    },
    getPlayersByTeam(team, index, teamId, teamName) {
      console.log("-----------------------------")
      var indexJoueur = team.indexOf("<a href=\"profil.php", (index + 1))
      var firstStep = team.substring(indexJoueur, (indexJoueur + 200))
      var id = firstStep.substring(firstStep.indexOf("id="), firstStep.indexOf("\">")).substring(3)
      var name = firstStep.substring(firstStep.indexOf("\">"), firstStep.indexOf("</a>")).substring(2)
      if (id === "7370") {
        name = "M@dinina"
      }
      this.isFrench(id).then(value => {
        if (value) {
          var found = this.frenchPlayers.find(player => {
            return player.name === name
          })
          if (found) {
            console.log("Le joueur est déjà présent dans la liste...")
          } else {
            this.frenchPlayers.push({
              id: id,
              name: name,
              team: teamName,
              logo: "assets/img/teams/logos/" + teamId + ".png"
            })
            console.log('ID nouveau joueur : ' + id)
            console.log('Nom nouveau joueur : ' + name)
          }
        } else {
          console.log("Le joueur " + name + " n'est pas français !")
        }
        var newIndex = team.indexOf("<a href=\"profil.php", (indexJoueur + 1))
        if (newIndex !== -1 && newIndex !== indexJoueur) {
          this.getPlayersByTeam(team, indexJoueur, teamId, teamName)
        } else if (this.lastTeam) {
          this.loading = false
        }
      })
    },
    isFrench(id) {
      return new Promise(resolve => {
        let isFrench = false
        axios.get('https://proleague.de/profil.php?id=' + id).then(response => {
          if (response.data.includes("France")) {
            isFrench = true
          }
          resolve(isFrench)
        })
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.home {
  width: 90%;
  height: 90%;
  display: flex;
  flex-direction: column;
  justify-content: space-evenly;

  .leagues {
    width: 100%;
    height: 20%;

    .head {
      display: flex;
      justify-content: space-between;
    }

    .leagues-block {
      display: flex;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
    }

    .search-btn {
      img {
        width: 5%;
        height: 5%;
      }
    }
  }

  .players {
    width: 100%;
    height: 70%;
    display: flex;
    flex-direction: column;

    .players-block {
      width: 100%;
      height: 90%;
      display: flex;
      flex-wrap: wrap;
      overflow-y: auto;

      .player {
        width: 15%;
        height: 10%;
        display: flex;
        margin: 10px;

        .profilepic {
          display: flex;
          align-items: center;
          height: 100%;
          margin-right: 10px;

          img {
            width: 60px;
            height: 60px;
          }
        }

        .data {
          display: flex;
          flex-direction: column;
          justify-content: center;

          .name {
            font-weight: bold;
            font-size: 20px;

            a {
              color: black;

              &:hover {
                color: black;
              }
            }
          }

          .team {
            span:first-child {
              font-weight: bold;
            }
          }
        }
      }
    }

  }
}
</style>
