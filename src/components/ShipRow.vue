<template >
  <div v-if="ship.grouped()" class="ship-group-div">
    <b-row class="ship-header-row">
      <b-col cols="6" class="ship-name">{{ ship.name }}</b-col>
      <b-col cols="4">
        <b-button block variant="primary" size="sm" v-on:click="purchaseShip(ship)" v-bind:disabled="disableBuy(ship)">New ({{ ship.cost }})</b-button>
      </b-col>
      <b-col cols="2"></b-col>
    </b-row>
    <div
        v-for="group in ship.groups()"
        v-bind:key="group.label">
      <b-row class="ship-group-row">
        <b-col cols="1">
          <div class="split-merge">
            <b-dropdown class="split-merge" size="sm" menu-class="split-merge-menu">
              <b-dropdown-header>{{group.label}}</b-dropdown-header>
              <b-dropdown-text v-if="hasNoCommands(ship, group)">Cannot split or merge</b-dropdown-text>
              <template v-if="ship.hasAvailableGroup()">
                <b-dropdown-item-button
                    v-for="i in group.count-1"
                    v-bind:key="splitKey(group, i)"
                    v-on:click="splitGroup(ship, group, i)"
                    >
                  Split out {{ i }}
                </b-dropdown-item-button>
              </template>
              <b-dropdown-divider v-if="canSplitAndMerge(ship, group)"></b-dropdown-divider>
              <b-dropdown-item-button
                  v-for="otherGroup in ship.mergableGroups(group)"
                  v-bind:key="mergeKey(group, otherGroup)"
                  v-on:click="mergeGroups(ship, group, otherGroup)">
                Merge into {{ otherGroup.label }}
              </b-dropdown-item-button>
            </b-dropdown>
          </div>
        </b-col>

        <b-col cols="5" class="ship-name">
          <b-badge variant="danger" v-if="allowReact(group)">R</b-badge>
          <b-badge variant="warning" v-if="canReact(group)">R</b-badge>
          {{group.label}}
          <b-badge variant="primary">{{group.count}}</b-badge>
        </b-col>
        <b-col cols="4">
          <b-button
              block
              variant="primary" 
              size="sm"
              v-if="upgradable(ship, group.label)"
              v-on:click="upgradeGroup(ship, group.label)"
              v-bind:disabled="disableUpgrade(ship, group.label)">
            Upgrade ({{ ship.upgradeCost(group.label) }})
          </b-button>
          <b-button 
              block
              variant="primary"
              size="sm"
              v-else
              v-on:click="purchaseShip(ship, group.label)"
              v-bind:disabled="disableBuy(ship, group.label)">
            Buy ({{ ship.cost }})
          </b-button>
        </b-col>
        <b-col cols="2">
          <b-button block variant="danger" size="sm" v-on:click="loseShip(ship, group.label)" v-bind:disabled="disableLose(ship, group.label)">Lose</b-button>
        </b-col>
      </b-row>
      <b-row class="ship-group-tech-row">
        <b-col cols="12" class="ship-group-tech"><span v-html="techInfoBar(group)"></span></b-col>
      </b-row>
    </div>
  </div>
  <b-row v-else align-v="center" align-h="center">
    <b-col cols="6" class="ship-name">{{ ship.name }} <b-badge :variant="shipBadgeVariant(ship)">{{ ship.currentCount }}</b-badge></b-col>
    <b-col cols="4">
      <b-button block variant="primary" size="sm" v-on:click="purchaseShip(ship)" v-bind:disabled="disableBuy(ship)">Buy ({{ ship.cost }})</b-button>
    </b-col>
    <b-col cols="2">
      <b-button block variant="danger" size="sm" v-on:click="loseShip(ship)" v-bind:disabled="disableLose(ship)">Lose</b-button>
    </b-col>
  </b-row>
</template>


<script>
import { PurchaseShipCommand, LoseShipCommand, UpgradeShipCommand, SplitGroupCommand, MergeGroupsCommand } from "../models/commands";
import { BIconCaretDownSquare } from 'bootstrap-vue';

export default {
  name: "ShipRow",
  components: { BIconCaretDownSquare },
  props: [ 'ship', 'techs', 'psheet' ],
  methods: {
    purchaseShip: function(ship, group) {
      if (!this.psheet.hasSubtractedMaintenancePoints()) {
        this.psheet._notifyWarning('You cannot purchase ships until after subtracting maintenance.');
        return;
      }

      if (!ship.requirementsMet(this.techs)) {
        missing = ship.missingRequirements(this.techs);
        warning = missing.map ( m => "You need " + m + " technology.").join("<br/>");
        this.psheet._notifyWarning(warning);
        return;
      }

      if (ship.cost > this.psheet.constructionPoints) {
        this.psheet._notifyWarning('You do not have enough CPs');
        return;
      };

      this.psheet._executeCommand(new PurchaseShipCommand(this.psheet, ship, group));
    },
    loseShip: function(ship, group) {
      if (!ship.canLose(group)) {
        this.psheet._notifyWarning("You don't have any more " + ship.name + "s to lose.");
        return;
      }

      this.psheet._executeCommand(new LoseShipCommand(this.psheet, ship, group));
    },
    upgradeGroup: function(ship, group) {
      if (!ship.canUpgrade(this.psheet.constructionPoints, this.techs, this.group)) {
        this.psheet._notifyWarning("You cannot upgrade that group");
        return;
      }

      this.psheet._executeCommand(new UpgradeShipCommand(this.psheet, ship, group));
    },
    splitGroup: function(ship, group, count) {
      if (!ship.hasAvailableGroup() || group.count <= 1) {
        this.psheet._notifyWarning("You cannot split that group.");
        return;
      }

      this.psheet._executeCommand(new SplitGroupCommand(this.psheet, ship, group.label, count)); 
    },
    mergeGroups: function(ship, fromGroup, toGroup) {
      this.psheet._executeCommand(new MergeGroupsCommand(this.psheet, ship, fromGroup.label, toGroup.label));
    },
    shipBadgeVariant: function(ship) {
      if (ship.currentCount > 0) {
        return 'success';
      }

      return 'secondary';
    },
    disableBuy: function(ship, groupName) {
      return !(this.psheet.hasSubtractedMaintenancePoints() && ship.canPurchase(this.psheet.constructionPoints, groupName))
    },
    disableLose: function(ship, groupName) {
      return !ship.canLose(groupName);
    },
    disableUpgrade: function(ship, groupName) {
      return !ship.canUpgrade(this.psheet.constructionPoints, this.techs, groupName);
    },
    upgradable: function(ship, groupName) {
      return ship.upgradable(this.techs, groupName);
    },
    techInfoBar: function(group) {
      return group.techUpgradeInfo(this.techs).map(tech => tech['tech'] + ':' + tech['level']).join(' | ');
    },
    splitKey: function(group, number) {
      return group.label + '-s-' + number;
    },
    mergeKey: function(fromGroup, toGroup) {
      return fromGroup.label + '-m-' + toGroup.label;
    },
    canSplitAndMerge: function(ship, group) {
      return ship.hasAvailableGroup() && group.count > 1 && ship.mergableGroups(group).length > 0;
    },
    hasNoCommands: function(ship, group) {
      return ship.mergableGroups(group).length === 0 && (!ship.hasAvailableGroup() || group.count <= 1);
    },
    canReact: function(group) {
      return this.psheet.findTechByTitle('Exploration').currentLevel >= 2 && group.canReact();
    },
    allowReact: function(group) {
      return group.allowReact();
    }
  }
}
</script>

<style scoped>
.ship-name {
  text-align: right;
  padding-right: 5px;
}

div.row {
  padding-top: 3px;
  padding-bottom: 3px;
}

.ship-group-div {
  border-bottom: 2px solid grey;
}

.ship-header-row .ship-name {
  text-align: center;
  font-weight: bold;
}

.ship-group-row {
  border-top: 1px dashed lightgrey;
}

.ship-group-tech {
  text-align: center;
  font-weight: bold;
}

.split-merge {
  position: inherit;
}

/deep/ .split-merge > .split-merge-menu {
  background-color: #D4F6FF;
  width: 250px;
}

@media only screen and (max-width: 400px) {
  div.row {
    margin-left: 0px;
    margin-right: 0px;
  }

  .row div {
    padding-left: 2px;
    padding-right: 2px;
  }

  .row div button {
    padding-left: 5px;
    padding-right: 5px;
    font-size: 0.85rem;
  }
}
</style>