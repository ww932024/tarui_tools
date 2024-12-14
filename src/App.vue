<script setup lang="ts">
import { reactive, ref } from "vue";
import zhCn from "element-plus/es/locale/lang/zh-cn";
import { useDark, useTransition } from "@vueuse/core";
import { ComponentSize, FormInstance, FormRules } from "element-plus";
import { fetch } from "@tauri-apps/plugin-http";

interface RuleForm {
  lotteryNum: string;
  id: string;
}

export interface OrderDetail {
  lotteryNum: string;
  isWinDes?: any;
  betNum: string;
  odd: string;
  winMoney: string;
  isWin: number;
  gameCategoryCode: string;
}

export interface Order {
  createBy: string;
  createTime: number;
  updateBy: string;
  updateTime: number;
  id: string;
  merchantCode: string;
  merchantName: string;
  betDate: string;
  orderNo: string;
  gameCategoryCode: string;
  chaseId?: any;
  isChase: number;
  gameCode: string;
  gameCodeName: string;
  gamePlayCode: string;
  gamePlayName: string;
  issueNo: string;
  lotteryNum: string;
  userId: string;
  userName: string;
  betCount: string;
  betMoney: number;
  betMoneyTotal: number;
  betTime: number;
  startSettleTime: number;
  settleTime: number;
  balance: number;
  multiple: number;
  odd: string;
  isWin: number;
  winCount: string;
  winMoney: number;
  validWin: number;
  state: number;
  betIp: string;
  currencyType: string;
  gamePlayCategoryCode: string;
  curOdd: string;
  gamePlayType: number;
  remark?: any;
  nums: string;
  numsName: string;
  routeName?: any;
  isIssueCancel: number;
  isChaseIssue: number;
  playGroupCode: string;
  playGroupName: string;
  profitMoney: number;
  validBetMoney: number;
  areaType: number;
  gameTranslateCode?: any;
}

export interface ResponseProps {
  orderDetail: OrderDetail[];
  order: Order;
}

const formSize = ref<ComponentSize>("default");
const ruleFormRef = ref<FormInstance>();
const loading = ref(false);
const tableData = ref<OrderDetail[]>([]);
const orderData = ref<Order>();
const ruleForm = reactive<RuleForm>({
  lotteryNum: "",
  id: "",
});
const params = reactive({
  page: 1,
  size: 10,
  jump: 0,
  list: [] as OrderDetail[][],
  originList: [] as OrderDetail[],
});

const statistic = ref([0, 0, 0]);
const output = useTransition<any>(statistic, {
  duration: 1500,
});

// 设置主题色，白色还是暗黑
useDark();

const rules = reactive<FormRules<RuleForm>>({
  // lotteryNum: [{ required: true, message: "请输入开奖号码", trigger: "blur" }],
  id: [{ required: true, message: "请输入用户注单号", trigger: "blur" }],
});

const submitForm = async (formEl: FormInstance | undefined) => {
  if (!formEl) return;
  await formEl.validate(async (valid, fields) => {
    params.page = 1;
    params.size = 10;
    if (valid) {
      loading.value = true;
      await getTableInfo();
      loading.value = false;
    } else {
      console.log("error submit!", fields);
    }
  });
};

const resetForm = (formEl: FormInstance | undefined) => {
  if (!formEl) return;
  formEl.resetFields();
  ruleForm.id = "";
  ruleForm.lotteryNum = "";
  orderData.value = undefined;
  loading.value = false;
  tableData.value = [];
  statistic.value = [0, 0, 0];
};

const chunkArray = (array: OrderDetail[], size: number) => {
  const result = [];
  for (let i = 0; i < array.length; i += size) {
    result.push(array.slice(i, i + size));
  }
  return result;
};

const getTableInfo = async () => {
  console.log("params---", ruleForm);
  const response = await fetch(
    "https://dev-gameweb.jx203.com/api/order-util/bettingOrder/betOrderDetail",
    {
      headers: {
        "Content-Type": "application/json",
      },
      method: "POST",
      // 大订单 1051832413321576475   小订单 1051832735716753958
      // 单个 1048296551056683432
      body: JSON.stringify(ruleForm),
    }
  );
  console.log("response----", response);
  if (response.status === 200) {
    const { data }: { data: ResponseProps } = await response.json();
    console.log("data----", data);
    if (typeof data === "string") {
      // @ts-ignore
      ElNotification.error({
        title: "温馨提示",
        message: "注单不存在",
      });
      return;
    }
    if (!data) {
      orderData.value = undefined;
      tableData.value = [];
      statistic.value = [0, 0, 0];
      return;
    }
    // 共betMoneyTotal组合，中奖组合，未中奖组合
    const isWin = data.orderDetail.filter((item) => item.isWin);
    const noWin = data.orderDetail.filter((item) => !item.isWin);
    statistic.value = [data.orderDetail.length, isWin.length, noWin.length];
    // 表格list
    const list = data.orderDetail.map((item, index) => ({
      ...item,
      id: index + 1,
    }));
    params.originList = list;
    console.log("list-----", params.list);
    params.list = chunkArray(list, params.size);
    orderData.value = data.order;
    tableData.value = params.list[0];
  } else {
    // @ts-ignore
    ElNotification.error({
      title: "温馨提示",
      message: "接口异常 500",
    });
  }
};

const pageChange = (page: number) => {
  console.log("page----", page);
  tableData.value = params.list[page - 1];
};

const sizeChange = (size: number) => {
  console.log("size----", size);
  params.page = 1;
  params.size = size;
  params.list = chunkArray(params.originList, size);
  tableData.value = params.list[0];
};
</script>

<template>
  <el-config-provider :locale="zhCn">
    <main class="container">
      <div class="center">
        <el-form
          ref="ruleFormRef"
          :model="ruleForm"
          :rules="rules"
          label-width="auto"
          :size="formSize"
          status-icon
        >
          <el-form-item label="用户注单号" prop="id">
            <el-input v-model="ruleForm.id" placeholder="请输入用户注单号" />
          </el-form-item>
          <el-form-item label="开奖号码" prop="lotteryNum">{{
            orderData?.lotteryNum
          }}</el-form-item>
          <el-form-item>
            <div style="width: 190px">
              <el-button
                type="primary"
                @click="submitForm(ruleFormRef)"
                :loading="loading"
              >
                开始计算
              </el-button>
              <el-button @click="resetForm(ruleFormRef)">重置</el-button>
            </div>
            <span class="total"
              >共计
              <span class="blue">{{ output[0].toFixed(0) }}</span>
              个组合，中奖组合：<span class="green">{{
                output[1].toFixed(0)
              }}</span
              >，未中奖组合：<span class="red">{{
                output[2].toFixed(0)
              }}</span></span
            >
          </el-form-item>
        </el-form>
        <el-table :data="tableData" border class="table" v-loading="loading">
          <template #empty>
            <p style="line-height: 360px">暂无数据</p>
          </template>
          <el-table-column align="center" prop="id" label="序号" width="60" />
          <el-table-column
            align="center"
            prop="isWin"
            label="是否中奖"
            width="90"
          >
            <template v-slot="scope">
              {{ scope.row.isWin ? "已中奖" : "未中奖" }}
            </template>
          </el-table-column>
          <el-table-column
            align="center"
            prop="betNum"
            class="lottery-num"
            label="买入组合/玩法"
            show-overflow-tooltip
          />
          <el-table-column align="center" prop="odd" label="赔率" width="100" />
          <el-table-column
            align="center"
            prop="winMoney"
            label="派彩金额"
            width="120"
          />
        </el-table>
        <el-pagination
          v-if="!loading"
          class="pagination"
          background
          layout="prev, pager, next, sizes, jumper, total"
          v-model:current-page="params.page"
          :page-sizes="[10, 20, 50, 100]"
          :total="statistic[0]"
          @current-change="pageChange"
          @size-change="sizeChange"
        />
      </div>
    </main>
  </el-config-provider>
</template>

<style>
.el-input__inner {
  box-shadow: none;
}

.el-table > .el-popper {
  width: 500px;
  text-align: center;
}

:root {
  font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
  font-size: 16px;
  line-height: 24px;
  font-weight: 400;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  -webkit-text-size-adjust: 100%;
}

.container {
  margin: 0;
  padding-top: 2vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.center {
  margin: 0 auto;
  width: 80%;
}

.table {
  width: 100%;
  height: 440px;
}

.total {
  margin-left: 20px;
  font-weight: bold;
}

.pagination {
  width: 100%;
  margin-top: 20px;
}

.blue {
  color: #409eff;
}

.red {
  color: #f56c6c;
}

.green {
  color: #67c23a;
}

.lottery-text {
  cursor: pointer;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}
</style>
