<template>
  <div class="container is-fluid" v-if="this.$cookie.get('Cookie')">

  <div class="container is-fluid">
  <h1 class="has-text-centered" v-if="!newname">Hej {{ this.name }}!</h1>
  <h1 class="has-text-centered" v-if="newname">Hej {{ this.newname }}!</h1>
  </div>

  <div class="container is-fluid">
  <h3>Kontaktinformation</h3>
   {{ this.realname }} | <a v-bind:href="emaillink">{{ this.email }}</a> | {{ this.address }}
   <button class="button" style="float:right; margin:0 5px;" @click="removeAccountWarning()">Radera konto</button>
  </div>

  <div class="container is-fluid">
  <h3>Ändra uppgifter <span class="minifont">(alla fält ska vara ifyllda! Logga ut efter sparade ändringar för att kunna låna!)</span></h3>
  <input v-model="newname" type="text" placeholder="Användarnamn">
  <input v-model="password" type="text" placeholder="Lösenord">
  <input v-model="email" type="text" placeholder="E-mail">
  <input v-model="realname" type="text" placeholder="Hela namn">
  <input v-model="address" type="text" placeholder="Adress">

  <input v-bind:value="updateUser" v-on:click="updateUserFunc" class="button" type="submit">
  </div>

  <div class="container is-fluid">
    <h3>Lånade böcker</h3>

  <div class="table_wrap">
    <table class="table is-hoverable is-fullwidth">
      <thead>
      <tr>
        <th>Titel</th>
        <th>Författare</th>
        <th>Lånedatum</th>
        <th>Utgångsdatum</th>
        <th>Dagar kvar</th>
        <th>Förläng lån</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="loan in loans">
        <td class="chosenBook" v-on:click="openBook">{{ loan.title }}</td>
        <td class="chosenBook" v-on:click="openBook">{{ loan.author }}</td>
        <td class="chosenBook" v-on:click="openBook">{{ loan.loanDate }}</td>
        <td class="chosenBook" v-on:click="openBook">{{ loan.returnDate }}</td>
        <td class="chosenBook" v-on:click="openBook"><countdown
          v-bind:return-date="loan.returnDate"
          ></countdown></td>
        <!-- <td><button class="button" @click="countDown(loan.returnDate)">{{count}}</button></td> -->
        <td><extend-button
          v-bind:book-id="loan.bookId"
          v-bind:user-id="loan.userId"
          v-bind:loan-date="loan.loanDate"
          v-on:added-to-loans="getUpdatedLoans"
          ></extend-button></td>
      </tr>
    </tbody>
    </table>
  </div>
 </div>
</div>
</template>

<script>
  // import UpdateUserButton from './updateUserButton.vue'
  import ExtendButton from './extendButton.vue'
  import Countdown from './countdown.vue'
  import router from "../router"
  import moment from 'moment'
  import { Dialog } from 'buefy/dist/components/dialog'

  export default {
  created() {
    this.fetchresult()
  },
    data() {
      return {
        updateUser: 'Ändra uppgifter',
        name: '',
        newname: '',
        password: '',
        email: '',
        realname: '',
        address: '',
        inloggad: true,
        userId: '',
        users: [],
        loans: [],
      }
    },
    components: {
      // 'update-user-button': UpdateUserButton,
      'extend-button': ExtendButton,
      'countdown': Countdown,
    },
    methods: {
      getUpdatedLoans(loans){
        //tar emot om något lån har förlängts (Sara)
        this.loans = loans
      },
      fetchresult() {
        fetch('http://localhost:3000/login')
        .then(response => response.json())
          .then (result => {
            //Hämtar namnet på usern som är inloggad utifrån userns cookie (Alex)
            this.name = result.find(value => value.token === this.$cookie.get('Cookie')).user
          })

          fetch('http://localhost:3000/users')
            // Hämtar kontaktinformation till den inloggade användaren (Maija/Alex)
            .then(response => response.json())
            .then (result => {
              this.realname = result.find(value => value.name === this.name ).realname
              this.email = result.find(value => value.name === this.name ).email
              this.address = result.find(value => value.name === this.name ).address
              this.userId = result.find(value => value.name === this.name ).id
            }).then(() => {
              //hämtar lån, med viss användare (sara, yasmine)
              fetch('http://localhost:3000/loans/' + this.userId)
                .then(response => response.json())
                .then (result => {
                  this.loans = result
                })
            })
        },
        // Låter user ta bort sitt konto. Får en varning först (Alex)
        removeAccountWarning() {
          this.$dialog.confirm({
                title:  'Ta bort kontot',
                message: 'Är du säker att du vill <b>ta bort</b> ditt konto? Du kan inte ångra detta.',
                confirmText: 'Ta bort',
                type: 'is-danger',
                hasIcon: true,
                onConfirm: () => {
                    this.removeAccount()
                    this.$toast.open('Ditt konto har tagits bort!')
                    }
            })
        },
        removeAccount() {
          // Låter user ta bort sitt konto (Alex)
          let cookie = {'Cookie': this.$cookie.get('Cookie')}
          fetch('http://localhost:3000/users', {
                method: 'DELETE',
                body: JSON.stringify(cookie),
                headers: {'Content-type': 'application/json'},
            }).then(function(response) {
                if (response.status === 200) {
                  fetch('http://localhost:3000/logout', {
                      method: 'POST',
                      body: JSON.stringify(cookie),
                      headers: {'Content-type': 'application/json'},
                  }).then(function(response) {
                      router.push("/")
                  })
                }else {
                  Dialog.alert('Något har gått fel, försök igen senare!')
                }
            })
            .then(function(result){
                console.log(result)
            })
            this.$cookie.delete('Cookie')
        },
        updateUserFunc() {
            // för att ändra den inloggade användares uppgifter (Maija):
            fetch('http://localhost:3000/users', {
                body: JSON.stringify( { oldname: this.name, newname: this.newname, password: this.password, email: this.email, realname: this.realname, address: this.address} ),
                headers: {
                  'Content-Type': 'application/json'
                },
                method: 'PUT'
              })
              .then(response => response.json())
              .then (result => {

                if(this.newname === "" || this.password === "") {
                    Dialog.alert({
                      title: 'Oops..',
                      message: 'Du måste fylla i användarnamn och lösenord!',
                      confirmText: 'OK',
                      type: 'is-primary',
                    })

                } else {
                    Dialog.alert({
                      title: 'Yay!',
                      message: 'Kontaktuppgifter är nu uppdaterade!',
                      confirmText: 'OK',
                      type: 'is-primary',
                    })
                }
                console.log('Uppdaterat!')
                result.send
              })
            },  // end of updateUserFunc()
        openBook(){
          Dialog.alert({
            message: 'Ladda ner boken för att kunna läsa den',
            confirmText: 'Ladda ner',
            type: 'is-primary',
            canCancel: true,
            cancelText: 'Abryt'
          })
        },
    }
  }
</script>


<style scoped>
.container {
  margin-top:20px;
}

.minifont {
  font-size:0.6em;
}
.chosenBook {
  cursor: pointer;
}

</style>
