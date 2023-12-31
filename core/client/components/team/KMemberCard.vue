<template>
  <KCard
    v-bind="$props"
    :header="header"
    :actions="itemActions"
    :bind-actions="false"
    :dense="dense"
    :options="{ nameField: 'profile.name', descriptionField: 'profile.description' }">
    <!--
      Card avatar
    -->
    <template v-slot:card-avatar>
      <KAvatar :subject="item" size="3.2rem"
        :options="{ nameField: 'profile.name', avatarField: 'profile.avatar' }"/>
    </template>
    <!--
      Card descriptions
    -->
    <template v-slot:card-description>
      <div class="row full-width justify-start" v-bind:class="{ 'q-py-xs': dense, 'q-py-sm': !dense }">
        {{ description }}
      </div>
    </template>
    <!--
      Card content
    -->
    <template v-slot:card-content>
      <!-- Tags section -->
      <KCardSection v-if="!item.expireAt"
        :title="$t('KMemberCard.TAGS_SECTION')"
        :actions="tagsActions"
        :context="$props"
        :dense="dense">
        <div v-if="hasTags" id="tags-section">
          <KChipsPane id="tags-pane" class="q-pa-sm" :remove="true" @chip-removed="onRemoveTag" :chips="tags" />
        </div>
        <div v-else>
          {{ $t('KMemberCard.NO_TAGS_LABEL')}}
        </div>
      </KCardSection>
      <!-- Groups section -->
      <KCardSection v-if="!item.expireAt"
        :title="$t('KMemberCard.GROUPS_SECTION')"
        :actions="groupsActions"
        :context="$props"
        :dense="dense"
      >
        <div v-if="hasGroups" id="groups-section" class="row justify-start items-center">
          <template v-for="(group, index) in groups" :key="groupKey(group)">
            <q-btn :id="groupButtonId(group)" flat small round color="primary">
              <q-avatar color="primary" text-color="white" size="2rem">{{ groupInitials(group) }}</q-avatar>
              <q-menu ref="popover">
                <q-toolbar inverted color="grey-7">
                  <span style="margin:8px">{{group.name}}</span>
                  <q-btn id="change-role-group" v-if="canChangeRoleInGroup(group)" flat round small @click="onChangeRoleInGroup(group), $refs.popover[index].hide()">
                    <q-icon :name="roleIcon(roleForGroup(group))" />
                  </q-btn>
                  <q-btn id="leave-group" v-if="canLeaveGroup(group)" flat round small @click="onLeaveGroup(group), $refs.popover[index].hide()">
                    <q-icon name="las la-minus-circle" />
                  </q-btn>
                </q-toolbar>
              </q-menu>
              <q-tooltip>{{ group.name }}</q-tooltip>
            </q-btn>
          </template>
        </div>
        <div v-else>
          {{ $t('KMemberCard.NO_GROUPS_LABEL')}}
        </div>
      </KCardSection>
      <!-- Expiration section -->
      <KCardSection v-if="item.expireAt" :dense="dense">
        <div class="row full-width justify-between items-center no-wrap">
          <span class="text-warning text-weight-bold" v-if="expireAt">
            {{$t('KMemberCard.EXPIRE_AT_LABEL')}} {{expireAt.toLocaleString()}}
          </span>
          <k-panel :content="expirationActions" />
        </div>
      </KCardSection>
    </template>
  </KCard>
</template>

<script>
import _ from 'lodash'
import { Dialog } from 'quasar'
import { baseItem } from '../../mixins'
import { getInitials, processIcon, isEmailValid } from '../../utils'
import { Roles, getRoleForOrganisation, getRoleForGroup, findGroupsWithRole } from '../../../common/permissions'
import { KCard, KCardSection } from '../collection'
import KPanel from '../KPanel.vue'
import KAvatar from '../KAvatar.vue'
import KChipsPane from '../KChipsPane.vue'

export default {
  components: {
    KCard,
    KCardSection,
    KAvatar,
    KPanel,
    KChipsPane
  },
  mixins: [baseItem],
  computed: {
    header () {
      const components = _.filter(this.itemActions, { scope: 'header' })
      const roleLabel = this.roleLabel(this.role)
      // It appears that due to reactivity role might not be available anymore when removing a member
      components.splice(0, 0, {
        id: 'role-badge', component: 'QBadge', label: (roleLabel ? this.$t(roleLabel) : ''), color: 'grey-7'
      }, { component: 'QSpace' })
      return components
    },
    avatar () {

    },
    role () {
      const role = getRoleForOrganisation(this.item, this.contextId)
      if (!_.isUndefined(role)) return Roles[role]
      return ''
    },
    tagsActions () {
      return _.filter(this.itemActions, { scope: 'tags' })
    },
    hasTags () {
      return !_.isEmpty(this.tags)
    },
    tags () {
      // Check for custom tags field
      let tags = _.get(this.item, this.options.tagsField || 'tags', [])
      // Filter tags from current context
      tags = _.filter(tags, { context: this.$store.get('context._id') })
      // Then process icons
      tags.forEach(tag => processIcon(tag))
      return tags
    },
    groupsActions () {
      return _.filter(this.itemActions, { scope: 'groups' })
    },
    hasGroups () {
      return !_.isEmpty(this.groups)
    },
    groups () {
      return _.filter(this.item.groups, { context: this.contextId })
    },
    expirationActions () {
      return _.filter(this.itemActions, { scope: 'expiration' })
    },
    expireAt () {
      return this.item.expireAt ? new Date(this.item.expireAt) : null
    },
    dense () {
      return this.$q.screen.lt.sm
    }
  },
  methods: {
    resendInvitation () {
      Dialog.create({
        title: this.$t('KMemberCard.RESEND_INVITATION_DIALOG_TITLE', { member: _.get(this.item, 'profile.name') }),
        message: this.$t('KMemberCard.RESEND_INVITATION_DIALOG_MESSAGE', { member: _.get(this.item, 'profile.name') }),
        html: true,
        prompt: {
          model: this.item.email,
          isValid: email => isEmailValid(email),
          type: 'email'
        },
        ok: {
          label: this.$t('OK'),
          flat: true
        },
        cancel: {
          label: this.$t('CANCEL'),
          flat: true
        }
      }).onOk(async (email) => {
        // Extract invitation data from user
        const user = _.pick(this.item, ['email', 'name', 'locale'])
        if (email) user.email = email
        // Add the sponsor information
        user.sponsor = {
          id: this.$store.get('user._id'),
          organisationId: this.contextId,
          roleGranted: getRoleForOrganisation(this.item, this.contextId)
        }
        // Remove the invited user
        const usersService = this.$api.getService('users')
        await usersService.remove(this.item._id)
        // Recreate invitation
        await usersService.create(user)
      })
    },
    removeMember () {
      Dialog.create({
        title: this.$t('KMemberCard.REMOVE_DIALOG_TITLE', { member: _.get(this.item, 'profile.name') }),
        message: this.$t('KMemberCard.REMOVE_DIALOG_MESSAGE', { member: _.get(this.item, 'profile.name') }),
        html: true,
        ok: {
          label: this.$t('OK'),
          flat: true
        },
        cancel: {
          label: this.$t('CANCEL'),
          flat: true
        }
      }).onOk(() => {
        const authorisationService = this.$api.getService('authorisations')
        authorisationService.remove(this.contextId, {
          query: {
            scope: 'organisations',
            subjects: this.item._id,
            subjectsService: this.contextId + '/members',
            resourcesService: 'organisations'
          }
        })
      })
    },
    onRemoveTag (removedTag) {
      Dialog.create({
        title: this.$t('KMemberCard.REMOVE_TAG_DIALOG_TITLE', { tag: removedTag.value }),
        message: this.$t('KMemberCard.REMOVE_TAG_DIALOG_MESSAGE', { tag: removedTag.value, member: _.get(this.item, 'profile.name') }),
        html: true,
        ok: {
          label: this.$t('OK'),
          flat: true
        },
        cancel: {
          label: this.$t('CANCEL'),
          flat: true
        }
      }).onOk(() => {
        const membersService = this.$api.getService('members', this.contextId)
        const tags = _.get(this.item, this.options.tagsField || 'tags', [])
        _.remove(tags, tag => tag._id === removedTag._id)
        membersService.patch(this.item._id, { tags })
      })
    },
    groupKey (group) {
      return this.item._id + group._id
    },
    groupButtonId (group) {
      return _.kebabCase(group.name + '-button')
    },
    groupInitials (group) {
      return getInitials(group.name)
    },
    roleIcon (role) {
      return this.roleIcons[role]
    },
    roleLabel (role) {
      return this.roleLabels[role]
    },
    roleForGroup (group) {
      const role = getRoleForGroup(this.item, this.contextId, group._id)
      if (!_.isUndefined(role)) return Roles[role]
      return ''
    },
    canJoinGroup () {
      const user = this.$store.get('user')
      // Organisation managers can manage all groups
      const role = getRoleForOrganisation(user, this.contextId)
      if (Roles[role] >= Roles.manager) return true
      // Group managers can manage members of their own groups so
      // can add members to a group if at least manager of one
      const groups = findGroupsWithRole(user, this.contextId, Roles.manager)
      // FIXME: we should also filter by the member groups so that if already added to all my groups we don't show the action
      return groups.length > 0
    },
    canChangeRoleInGroup (group) {
      const memberRole = getRoleForOrganisation(this.item, this.contextId)
      if (Roles[memberRole] >= Roles.manager) return false
      const user = this.$store.get('user')
      // Organisation managers can manage all groups
      const userRole = getRoleForOrganisation(user, this.contextId)
      if (Roles[userRole] >= Roles.manager) return true
      // Group managers can manage members of their own groups
      else return this.$can('create', 'authorisations', null, { resource: group._id, permissions: 'member' })
    },
    canLeaveGroup (group) {
      const user = this.$store.get('user')
      // Organisation managers can manage all groups
      const role = getRoleForOrganisation(user, this.contextId)
      if (Roles[role] >= Roles.manager) return true
      // Group managers can manage members of their own groups
      else return this.$can('remove', 'authorisations', null, { resource: group._id, permissions: 'member' })
    },
    onChangeRoleInGroup (group) {
      this.$router.push({
        name: 'edit-member-role',
        params: {
          contextId: this.contextId,
          objectId: this.item._id,
          resource: { id: group._id, scope: 'groups', service: this.contextId + '/groups' }
        }
      })
    },
    onLeaveGroup (group) {
      Dialog.create({
        title: this.$t('KMemberCard.LEAVE_GROUP_DIALOG_TITLE', { group: group.name }),
        message: this.$t('KMemberCard.LEAVE_GROUP_DIALOG_MESSAGE', { group: group.name, member: _.get(this.item, 'profile.name') }),
        html: true,
        ok: {
          label: this.$t('OK'),
          flat: true
        },
        cancel: {
          label: this.$t('CANCEL'),
          flat: true
        }
      }).onOk(() => {
        const authorisationService = this.$api.getService('authorisations')
        authorisationService.remove(group._id, {
          query: {
            scope: 'groups',
            subjects: this.item._id,
            subjectsService: this.contextId + '/members',
            resourcesService: this.contextId + '/groups'
          }
        })
      })
    }
  },
  created () {
    // Load the role configuration
    this.roleIcons = this.$config('roles.icons')
    this.roleLabels = this.$config('roles.labels')
  }
}
</script>
