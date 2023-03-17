<template>
  <div class="tableInput" v-if=istable>
    <textarea class="inp" v-model="input"> </textarea>
    <input class="btn" type="button" value="Ввод" @click=settable>
  </div>

  <table v-if="!istable">
    <tr  v-for="stroke in filteredTable" :key="stroke">
      <th  v-for="val in stroke" :key="val">{{val}}</th>
    </tr>
  </table>

  <table v-if="!istable" style="table-layout: fixed; width: 300px;">
    <tr  v-for="stroke in parettotable" :key="stroke">
      <th  v-for="val in stroke" :key="val">{{val}}</th>
    </tr>
  </table>

  <div v-if="!istable" style="margin-top: 20px; margin-bottom: 30px">Парето-оптимальное множество определено альтернативами {{parettoNums}}</div>

  <div v-for="(filt, index) in sliderArr" :key="filt" class="slidersDiv">
    {{filt.name}}
    <input type="range" :min="filt.min" :max="filt.max" step="0.01" v-model="this.filters[index]"> {{this.filters[index]}}
  </div>

  <div v-if="!istable">
    сортировать по:
    <select v-model="this.sortBy">
      <option v-for="(val, i) in params" :key="val" :value="i">{{val}}</option>
    </select>
  </div>

</template>

<script>

export default {
  name: 'App',
  data() {
    return {
      'table': [],
      'parettotable': [],
      'parettoNums': [],
      'filters': [],
      'sliderArr': [],
      'params': [],
      'sortBy': 0,
      'input': '      | № |           Название         | Цена(руб)(-) | Вес(кг)(+) | Время отклика(мс)(-) | Количество клавиш(+) |\n' +
          '      |:-:|----------------------------|:------------:|:----------:|:--------------------:|:--------------------:|\n' +
          '      | 1 | Razer Huntsman V2 TKL      |     12760    |   0.748    | 0.125                | 87                   |\n' +
          '      | 2 | A4TECH Bloody B140N        |     1940     |   0.870    | 1                    | 104                  |\n' +
          '      | 3 | Oklick 920G IRON EDGE      |     3999     |   1.400    | 1                    | 104                  |\n' +
          '      | 4 | A4TECH Bloody B820R        |     4620     |   0.825    | 0.2                  | 104                  |\n' +
          '      | 5 | A4TECH Bloody B314         |     2399     |   1.020    | 0.2                  | 113                  |\n' +
          '      | 6 | A4TECH Bloody B500N        |     2499     |   0.800    | 1                    | 104                  |\n' +
          '      | 7 | Red Square Keyrox TKL g3ms |     5499     |   0.880    | 1                    | 87                   |\n' +
          '      | 8 | HIPER MK-1 SPIKE           |     3999     |   0.950    | 1                    | 61                   |'
    }
  },
  computed: {
    istable() { return this.table.length === 0 },
    filteredTable() {
      const result = [this.table[0]];
      for (let i = 1; i < this.table.length; i++) {
        let addCurrent = true;
        for (let i2 = 2; i2 < this.table[0].length; i2++) {
          let char = this.table[0][i2].charAt (this.table[0][i2].length - 2);
          if(  ( char=='+' &&  (this.table[i][i2] < this.filters[i2-2]) )    ||   (char=='-' && (this.table[i][i2] > this.filters[i2-2]) )  ) {addCurrent = false; break;}
        }
        if(addCurrent) result.push(this.table[i]);
      }
      if(this.sortBy == 0) return result;
      const sorted = this.sortTableByColumn(result, this.sortBy+1, result[0][this.sortBy+1].charAt(result[0][this.sortBy+1].length-2) == '-');
      return sorted;
    }
  },
  methods: {
    settable () {
      this.table = this.tableResolver (this.input);
      this.parettotable = this.addTableNumbering(this.paretto(this.table));
      this.parettoNums = this.extractNums(this.paretto(this.table));
      this.sliderArr = this.sliders(this.table);
      this.params = this.parcalc();
    },

    sortTableByColumn(table, column, ascending = true) {
      // Проверяем, что указанный столбец существует в таблице
      if (column >= table[0].length) {
        throw new Error(`Column ${column} does not exist in table`);
      }

      // Копируем исходный массив, чтобы не менять его
      const sortedTable = [...table];

      // Определяем порядок сортировки
      const sortOrder = ascending ? 1 : -1;

      // Сортируем копию массива по указанному столбцу
      sortedTable.sort((a, b) => {
        if (a[column] < b[column]) {
          return -1 * sortOrder;
        } else if (a[column] > b[column]) {
          return 1 * sortOrder;
        } else {
          return 0;
        }
      });

      // Возвращаем отсортированный массив
      return sortedTable;
    },

    sliders(table) {
      const res = [];
      for(let i = 2; i < table[0].length; i++) {
        let temp = {name: table[0][i] ,min: 99999, max: 0}
        for(let ind = 0; ind < table.length; ind++) {
          if(table[ind][i] > temp.max) temp.max = table[ind][i]+1;
          if(table[ind][i] < temp.min) temp.min = table[ind][i]-1;
        }
        this.filters[i-2] = temp.name.charAt(temp.name.length-2) == '+'? temp.min : temp.max;
        res.push(temp)
      }
      return res;
    },
    parcalc() {
      const awns = ['Не сортировать'];
      for(let i = 2; i < this.table[0].length; i++) {
        awns.push(this.table[0][i]);
      }
      return awns;
    },
    paretto (resolved) {
      const out = [];
      for (let i1 = 0; i1 < resolved.length - 1; i1++) {
        let temp = [];
        for (let i2 = 0; i2 < resolved.length - 1; i2++) {
          if (i2 >= i1) temp.push ('x');
          else temp.push (this.compare (i1, i2, resolved));
        }
        out.push (temp);
      }
      return out;
    },

    compare (i1, i2, resolved) {
      let c1 = resolved[i1 + 1];
      let c2 = resolved[i2 + 1];
      let p = 0;  // Положительно - в сторону первого сравниваемого
      let eq = 0;
      for (let i = 2; i < resolved[0].length; i++) {
        c1[i] = parseFloat (c1[i]);
        c2[i] = parseFloat (c2[i]);
        if (c1[i] == c2[i]) eq++;
        else {
          if (resolved[0][i].charAt (resolved[0][i].length - 2) == '+') {
            if (c1[i] > c2[i]) p++;
            else p--;
          } else {
            if (c1[i] < c2[i]) p++;
            else p--;
          }
        }
      }
      if ((p + eq) == resolved[0].length - 2) return 'А' + (i1 + 1);
      if ((p - eq) == -(resolved[0].length - 2)) return 'А' + (i2 + 1);
      return 'н';
    },
    borderCut (table, parametr, value) {
      const out = [];
      out.push (table[0]);
      if (parametr < 2) {
        console.error ('Не может быть параметр 1 или 0!');
        return null;
      }
      const ch = table[0][parametr].charAt (table[0][parametr].length - 2);
      for (let i = 1; i < table.length; i++) {
        if (ch == "+") {
          if (table[i][parametr] >= value) out.push (table[i]);
        } else {
          if (table[i][parametr] <= value) out.push (table[i]);
        }
      }
      return out;
    },

    tableResolver (t) {
      t = t.replace (/ /g, '');
      const awns = [];
      t.split ('\n').forEach ((v, i) => {
        let tempArr = [];
        if (i != 1) {
          v.split ('|').forEach ((val) => {
            if (val) {
              tempArr.push (val);
            }
          })
          awns.push (tempArr)
        }
      })
      return awns;
    },

    extractNums: function (table) {
      var out = [];
      for (let i1 = 0; i1 < table.length; i1++) for (let i2 = 0; i2 < table[0].length; i2++) {
        let a = table[i1][i2];
        if (a != 'н' && a != 'x' && out.indexOf (a) === -1) out.push (a)
      }
      return out.sort ();
    },
    addTableNumbering(arr) {
      const newArr = [];
      newArr.push(['', ...Array.from({ length: arr[0].length }, (_, i) => i + 1)]);
      arr.forEach((row, index) => {
        const newRow = [index + 1, ...row];
        newArr.push(newRow);
      });
      return newArr;
    }
  }
}
</script>

<style>
.inp {
  position: absolute;
  margin-left: 25%;
  margin-top: 10%;
  height: 40%;
  width: 50%;
}
.btn {
  position: absolute;
  margin-left: 45%;
  margin-top: 35%;
  height: 4%;
  width: 8%;
}
table {
  border: 1px solid grey;
  margin-top: 30px;
  margin-left: 10px;
}
/* границы ячеек первого ряда таблицы */
th {
  border: 1px solid grey;
}
/* границы ячеек тела таблицы */
td {
  border: 1px solid grey;
}
</style>
