<script lang="ts" setup>
import { inject, ref, onMounted } from "vue"

import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome"
import { library } from "@fortawesome/fontawesome-svg-core"
import { fas } from "@fortawesome/free-solid-svg-icons"
library.add(fas)

import { time_ago } from "../utils"
import MainLayout from "../layouts/MainLayout.vue"
import Plan from "@/components/Plan.vue"
import idb from "../idb"

const setPlanData = inject("setPlanData")
const url = new URL(window.location.href)
const planInput = ref<string>("")
const queryInput = ref<string>("")
const queryName = ref<string>("")
const slowQueryMode = ref<string>("requestId")
const pg3SlowQueryExplainUrl = ref<string>("")
const pg3SlowQueryAnalyze = ref<boolean | null>(
  url.searchParams.get("analyze") === "true"
)
const pg3SlowQueryTimeout = ref<string | null>(url.searchParams.get("timeout"))
const draggingPlan = ref<boolean>(false)
const draggingQuery = ref<boolean>(false)
const savedPlans = ref<Plan[]>()

function handleSlowQueryModeChange() {
  if (slowQueryMode.value === "requestId") {
    pg3SlowQueryExplainUrl.value = `https://${
      window.location.host
    }/api/v0/system/stats/slow_queries/${
      url.searchParams.get("requestId") ?? "<requestId>"
    }/_explain`
  } else {
    pg3SlowQueryExplainUrl.value = `https://${
      window.location.host
    }/api/v0/system/stats/${
      url.searchParams.get("projectId") ?? "<projectId>"
    }/slow_queries/${
      url.searchParams.get("statementHash") ?? "<pg3StatementHash>"
    }/_explain`
  }
}

function fetchPlanFromPG3() {
  if (
    pg3SlowQueryExplainUrl.value.indexOf("<projectId>") === -1 &&
    pg3SlowQueryExplainUrl.value.indexOf("<pg3StatementHash>") === -1 &&
    pg3SlowQueryExplainUrl.value.indexOf("<requestId>") === -1
  ) {
    var requestHeaders = new Headers()
    requestHeaders.append("Accept", "application/json")

    var requestOptions = {
      method: "GET",
      headers: requestHeaders,
      redirect: "follow",
      credentials: "same-origin",
    }

    planInput.value = "Fetching plan from PG3..."
    var fetchUrl = new URL(pg3SlowQueryExplainUrl.value)
    fetchUrl.search = new URLSearchParams({
      analyze: pg3SlowQueryAnalyze.value ? "true" : "false",
      timeout: pg3SlowQueryTimeout.value ? pg3SlowQueryTimeout.value : "15s",
    }).toString()

    fetch(fetchUrl, requestOptions)
      .then((response) => response.text())
      .then((result) => {
        planInput.value = result
      })
      .catch((error) => {
        planInput.value = `Error when fetching explain from PG3 ${pg3SlowQueryExplainUrl.value} ${error}`
        console.log(error)
      })
  }
}

function submitPlan() {
  let newPlan: Plan = ["", "", ""]
  newPlan[0] =
    queryName.value ||
    "New Plan - " +
      new Date().toLocaleString("en-US", {
        dateStyle: "medium",
        timeStyle: "medium",
      })
  newPlan[1] = planInput.value
  newPlan[2] = queryInput.value
  newPlan[3] = new Date().toISOString()
  savePlanData(newPlan)

  setPlanData(...newPlan)
}

async function savePlanData(sample: Plan) {
  await idb.savePlan(sample)
}

onMounted(() => {
  const textAreas = document.getElementsByTagName("textarea")
  Array.prototype.forEach.call(textAreas, (elem: HTMLInputElement) => {
    elem.placeholder = elem.placeholder.replace(/\\n/g, "\n")
  })
  const noHashURL = window.location.href.replace(/#.*$/, "")
  window.history.replaceState("", document.title, noHashURL)
  handleSlowQueryModeChange()
  loadPlans()
  fetchPlanFromPG3()
})

async function loadPlans() {
  const plans = await idb.getPlans()
  savedPlans.value = plans.slice().reverse()
}

function loadPlan(plan?: Plan) {
  if (!plan) {
    return
  }

  queryName.value = plan[0]
  planInput.value = plan[1]
  queryInput.value = plan[2]
}

function openPlan(plan: Plan) {
  setPlanData(plan[0], plan[1], plan[2])
}

function editPlan(plan: Plan) {
  loadPlan(plan)
}

async function deletePlan(plan: Plan) {
  await idb.deletePlan(plan)
  loadPlans()
}

function handleDrop(event: DragEvent) {
  const input = event.srcElement
  if (!(input instanceof HTMLTextAreaElement)) {
    return
  }
  draggingPlan.value = false
  draggingQuery.value = false
  if (!event.dataTransfer) {
    return
  }
  const file = event.dataTransfer.files[0]
  const reader = new FileReader()
  reader.onload = () => {
    if (reader.result instanceof ArrayBuffer) {
      return
    }
    input.value = reader.result || ""
    input.dispatchEvent(new Event("input"))
  }
  reader.readAsText(file)
}
</script>

<template>
  <main-layout>
    <div class="container">
      <div class="alert alert-warning">
        <h3>PostgreSQL execution plan visualizer</h3>
      </div>
      <div class="row">
        <div class="col-sm-7">
          <div class="form-group">
            <label for="pg3SlowQueryExplainUrl"> Fetch plan from PG3 by </label>
            <p />
            <input
              type="radio"
              id="byRequestId"
              value="requestId"
              v-model="slowQueryMode"
              @change="handleSlowQueryModeChange"
            />
            <label for="one">requestId</label>
            <input
              type="radio"
              id="byStatementHash"
              value="statementHash"
              v-model="slowQueryMode"
              @change="handleSlowQueryModeChange"
            />
            <label for="two">statementHash</label>
            <p />
            <input
              type="text"
              class="form-control"
              id="pg3SlowQueryExplainUrl"
              v-model="pg3SlowQueryExplainUrl"
              placeholder="pg3 slow query url"
            />
            <p />

            <div class="row">
              <div class="col-sm-2">
                <div class="custom-control custom-checkbox">
                  <label class="custom-control-label" for="pg3SlowQueryAnalyze"
                    >Analyze</label
                  >
                  <input
                    type="checkbox"
                    class="custom-control-input"
                    id="pg3SlowQueryAnalyze"
                    v-model="pg3SlowQueryAnalyze"
                  />
                </div>
              </div>
              <div class="col-sm-1">
                <label for="pg3SlowQueryTimeout"> Timeout </label>
              </div>
              <div class="col-sm-4">
                <input
                  type="text"
                  class="form-control"
                  style="width: 100%"
                  id="pg3SlowQueryTimeout"
                  v-model="pg3SlowQueryTimeout"
                  placeholder="eg: 30s or 2m (default: 15s)"
                />
              </div>
            </div>
            <p />

            <button
              type="button"
              @click="fetchPlanFromPG3()"
              class="btn btn-primary"
            >
              Fetch plan from PG3
            </button>
            <p />
          </div>
          <form v-on:submit.prevent="submitPlan">
            <div class="mb-3">
              <label for="planInput" class="form-label">
                Plan <span class="small text-muted">(text or JSON)</span>
              </label>
              <textarea
                :class="['form-control', draggingPlan ? 'dropzone-over' : '']"
                id="planInput"
                rows="8"
                v-model="planInput"
                @dragenter="draggingPlan = true"
                @dragleave="draggingPlan = false"
                @drop.prevent="handleDrop"
                placeholder="Paste execution plan\nOr drop a file"
              >
              </textarea>
            </div>
            <div class="mb-3">
              <label for="queryInput" class="form-label">
                Query <span class="small text-muted">(optional)</span>
              </label>
              <textarea
                :class="['form-control', draggingQuery ? 'dropzone-over' : '']"
                id="queryInput"
                rows="8"
                v-model="queryInput"
                @dragenter="draggingQuery = true"
                @dragleave="draggingQuery = false"
                @drop.prevent="handleDrop"
                placeholder="Paste corresponding SQL query\nOr drop a file"
              >
              </textarea>
            </div>
            <div class="mb-3">
              <label for="queryName" class="form-label">
                Plan Name <span class="small text-muted">(optional)</span>
              </label>
              <input
                type="text"
                class="form-control"
                id="queryName"
                v-model="queryName"
                placeholder="Name for the plan"
              />
            </div>
            <button type="submit" target="_blank" class="btn btn-primary">
              Visualize Plan
            </button>
          </form>
        </div>
        <div class="col-sm-5 mb-4 mt-4 mt-md-0">
          <div class="mb-2">Saved Plans</div>
          <ul class="list-group" v-cloak>
            <li
              class="list-group-item px-2 py-1"
              v-for="plan in savedPlans"
              :key="plan.id"
            >
              <div class="row">
                <div class="col">
                  <button
                    class="btn btn-sm btn-outline-secondary py-0 ms-1 float-end"
                    title="Remove plan from list"
                    v-on:click.prevent="deletePlan(plan)"
                  >
                    <font-awesome-icon icon="trash"></font-awesome-icon>
                  </button>
                  <button
                    class="btn btn-sm btn-outline-secondary py-0 float-end"
                    title="Edit plan details"
                    v-on:click.prevent="editPlan(plan)"
                  >
                    <font-awesome-icon icon="edit"></font-awesome-icon>
                  </button>
                  <a
                    v-on:click.prevent="openPlan(plan)"
                    href=""
                    title="Open the plan details"
                  >
                    {{ plan[0] }}
                  </a>
                </div>
              </div>
              <div class="row">
                <div class="col">
                  <small class="text-muted">
                    created
                    <span :title="plan[3]?.toString()">
                      {{ time_ago(plan[3]) }}
                    </span>
                  </small>
                </div>
                <div class="col-6 text-end"></div>
              </div>
            </li>
          </ul>
          <p class="text-muted text-center" v-if="!savedPlans?.length" v-cloak>
            <em> You haven't saved any plan yet.</em>
          </p>
        </div>
      </div>
    </div>
  </main-layout>
</template>
