<template>
	<div class="flex flex-col space-y-[8px]">
		<div class="text-[12px] text-gray-600">
			{{ fieldMetaInfo?.label }}
		</div>
		<div>
			<!-- Data field -->
			<Input
				v-if="fieldMetaInfo?.fieldtype === 'Data'"
				type="text"
				@change="
					(val) => {
						onInput(val)
					}
				"
				:value="fieldValue"
				:debounce="300"
				:disabled="!editable"
			/>
			<!-- Link / Select field -->
			<Autocomplete
				v-else
				:placeholder="`Select ${fieldMetaInfo?.label.toLowerCase()}`"
				class="rounded-md"
				:class="{
					'border-red-500 border': triggerValidationError,
				}"
				:value="fieldValue"
				@change="
					(val) => {
						onInput(val?.value)
					}
				"
				:resourceOptions="getResourceOptions()"
				:searchable="editable"
			/>
		</div>
	</div>
</template>

<script>
import Autocomplete from "@/components/global/Autocomplete.vue"
import { inject, computed } from "vue"

export default {
	name: "TicketField",
	props: {
		ticketId: {
			type: Number,
			required: true,
		},
		fieldname: {
			type: String,
			required: true,
		},
		editable: {
			type: Boolean,
			default: true,
		},
		validate: {
			type: Function,
			default: function () {
				return true
			},
		},
		triggerValidationError: {
			type: Boolean,
			default: false,
		},
		// TODO: use a prop to trigger mandatory field validation errors: use case (ticket type should be set before changing ticket status)
	},
	components: {
		Autocomplete,
	},
	setup(props) {
		const $socket = inject("$socket")
		const $tickets = inject("$tickets")
		const ticket = computed(() => {
			return $tickets.get({ ticketId: props.ticketId }, { $socket }).value
		})

		return {
			ticket,
		}
	},
	computed: {
		fieldMetaInfo() {
			return this.$resources.fieldMetaInfo.data || null
		},
		fieldValue() {
			if (this.fieldname == "_assign") {
				return this.$resources.getAssignee.data?.agent_name || null
			}
			return this.ticket?.[this.fieldname] || null
		},
	},
	mounted() {
		if (this.fieldname == "_assign") {
			this.$socket.on("ticket_assignee_update", (data) => {
				if (data.ticket_id == this.ticket.name) {
					this.$resources.getAssignee.fetch()
				}
			})
		}
	},
	resources: {
		fieldMetaInfo() {
			// field can be a custom or a standard field
			// field type : Data, Link, Select
			// field label
			// field permissions: read, write (based on is agent / customer)
			return {
				method: "frappedesk.api.ticket.get_field_meta_info",
				params: {
					fieldname: this.fieldname,
				},
				auto: true,
			}
		},
		getAssignee() {
			// TODO: this is a temparary fix, should be done in a better way
			if (this.fieldname !== "_assign") return
			return {
				method: "frappedesk.api.ticket.get_assignee",
				params: {
					ticket_id: this.ticketId,
				},
				auto: true,
			}
		},
	},
	methods: {
		onInput(value) {
			if (this.validate()) {
				this.$tickets
					.set(this.ticketId, this.fieldname, value)
					.then(() => {
						this.$toast({
							title: "Ticket updated successfully.",
							appearance: "success",
							customIcon: "circle-check",
						})
					})
			} else {
				console.log("validation failed")
				this.$toast({
					title: "Please fill all mandatory fields.",
					customIcon: "circle-fail",
					appearance: "danger",
				})
			}
		},
		getResourceOptions() {
			if (!(this.editable || this.fieldMetaInfo)) return
			switch (this.fieldMetaInfo?.fieldtype) {
				case "Link":
					return {
						method: "frappe.client.get_list",
						inputMap: (query) => {
							return {
								doctype: this.fieldMetaInfo?.options,
								pluck: "name",
								filters: [["name", "like", `%${query}%`]],
							}
						},
						responseMap: (res) => {
							return res.map((d) => {
								return {
									label: d.name,
									value: d.name,
								}
							})
						},
					}
				case "Select":
					return {
						method: "frappedesk.api.general.get_filtered_select_field_options",
						inputMap: (query) => {
							return {
								doctype: "Ticket",
								fieldname: this.fieldMetaInfo?.fieldname,
								query,
							}
						},
						responseMap: (res) => {
							return res.map((d) => {
								return {
									label: d,
									value: d,
								}
							})
						},
					}
				default:
					return null
			}
		},
	},
}
</script>
