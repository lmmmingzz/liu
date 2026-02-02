flowchart TD
  %% =========================
  %% Entry
  %% =========================
  A0([__main__]) --> A1[读取实例目录 path]
  A1 --> A2[遍历 instance_set: file, count_file]
  A2 --> A3[调用 run_instance(file, count_file, event_id,\n enable_tou_price / enable_tou_shift / enable_ls_after_break)]
  
  %% =========================
  %% run_instance
  %% =========================
  subgraph S1[run_instance]
    B0[read_fuzzy_problem(file)\n-> Processing_time, J_num, M_num, J, O_num,\nprocessing_power, standby_power, events] --> B1{events 是否存在?}
    B1 -- 否 --> B1e[[raise ValueError: no #EVENT]]
    B1 -- 是 --> B2{event_id 合法?}
    B2 -- 否 --> B2e[[raise ValueError: out of range]]
    B2 -- 是 --> B3[取故障事件 break_event=(break_machine, fuzzy_break_start, fuzzy_repair_time)]

    B3 --> B4[初始化 NNIA(O_num, processing_power, standby_power)\napply_cfg_to_nnia(DOE_CFG)\n设置 enable_tou_price/enable_tou_shift 开关]
    B4 --> B5[Encode(Processing_time, Pop_size, J, J_num, M_num)]
    B5 --> B6[Order_Matrix(Processing_time)\n-> Machines_available, Times_available\n写入 nnia]
    
    B6 --> B7{enable_tou_price?}
    B7 -- 是 --> B8[构建 price_fn = smooth_tou_price_fn(...)\n构建 tou_integrator = ToUIntegrator(price_fn, period, dt)\n写入 nnia.tou_integrator]
    B7 -- 否 --> B9[nnia.tou_integrator=None\n(注意 Decode/Machine 需兼容)]
    
    B8 --> B10[随机初始化种群 B = e.Random_initial()]
    B9 --> B10
  end

  A3 --> B0

  %% =========================
  %% Main Evolution Loop
  %% =========================
  subgraph S2[for it in range(Max_Iterations)]
    C0[Step0: nnia.fitness(B, break_event, local_search=False)\n返回 B_makespan, B_TEC, B_STB] --> C1{enable_ls_after_break?}
    
    %% elite local search
    C1 -- 是 --> C2[Step0.5: 选精英 K=10%\n按 0.4*ms+0.3*tec+0.3*stb 打分] --> C3[nnia.fitness(elite, local_search=True,\nls_trials=2, lamarck_p)\n返回 elite2 与目标值]
    C3 --> C4[用 elite2 覆盖回 B 对应位置\n同步更新 B_makespan/B_TEC/B_STB]
    C1 -- 否 --> C5[跳过精英局部搜索]
    C4 --> C6
    C5 --> C6
    
    %% env selection
    C6[Step1: 非支配排序 fast_non_dominated_sort_N(B_*)] --> C7[拥挤距离选择 crowding_distance_selection_N\n得到父代 P (size=Pop_size)]
    
    %% mating pool via tournament
    C7 --> C8[Step1.5: 对 P 再做非支配排序\n计算 rank + crowding distance]
    C8 --> C9[二元锦标赛生成 mating_pool\n(pool_size = Active_Pop_size)]
    
    %% crossover / mutation
    C9 --> C10[Step2: 交叉/变异生成 offspring\n- Pc: machine_cross 或 operation_cross\n- Pm: machine_variation 或 operation_variation\n- 异常则 fallback(_safe_ms_cross/_safe_os_mutate)]
    
    %% merge
    C10 --> C11[Step3: 合并 B = vstack(P, offspring)\n进入下一次迭代]
  end

  B10 --> C0

  %% =========================
  %% Fitness internals (NNIA.fitness)
  %% =========================
  subgraph S3[nnia.fitness(population, break_event, local_search?)]
    D0[检查 enable_tou_price & tou_integrator\n构建 tou_params, tou_int] --> D1[对每个个体 chs:\ncache key = (break_event_hash, mode, hash(chs))]
    D1 --> D2{cache 命中?}
    D2 -- 是 --> D3[直接返回缓存的 (ms, tec, stb)\n若 return_pop: 返回缓存 best_chs]
    D2 -- 否 --> D4[Decode: d0.decode(chs)\n得到 old schedule & base TEC]
    
    D4 --> D5{break_event 是否为 None?}
    D5 -- 是 --> D6[无故障: ms=d0.makespan\ntec=d0.TEC\nstb=0\n写缓存并返回]
    
    D5 -- 否 --> D7[ReDecode: rd0.redecode(chs)\n考虑故障右移规则\n得到 ms0, tec0]
    D7 --> D8[compute_stability(d0, rd0)\n得到 stb0 (shift 前计算)]
    
    %% local search branch
    D8 --> D9{local_search?}
    D9 -- 是 --> D10[受扰定位 get_affected_positions...\n得到 affected_ops / affected_ms_sites / affected_os_indices / rho]
    D10 --> D11[按 rho 自适应 trials\n对受扰位点做 local_mutate_ms + local_mutate_os]
    D11 --> D12[对候选 cand 重新 ReDecode\n比较 _local_score 选 best_chs/best_rd]
    D9 -- 否 --> D13[best = (chs, rd0)]
    D12 --> D14
    D13 --> D14
    
    %% ToU-shift branch
    D14 --> D15{enable_tou_price AND enable_tou_shift\nAND enable_tou_shift_new?}
    D15 -- 是 --> D16[仅对 best_rd 做 apply_tou_shift(...)\n重算 best_tec = best_rd.Tou_Cost_Fast()]
    D15 -- 否 --> D17[不移动工序策略\nbest_tec 保持不变]
    
    %% Lamarckian write-back
    D16 --> D18[若 local_search 且 rand < lamarck_p:\n写回 best_chs 到 final_chs]
    D17 --> D18
    D18 --> D19[输出 best_ms/best_tec/best_stb\n并写入 eval_cache]
  end

  %% connect fitness block (conceptual)
  C0 -. 调用 .-> D0
  C3 -. 调用 .-> D0

  %% =========================
  %% After iterations: final evaluation & outputs
  %% =========================
  subgraph S4[Repeat 收尾与输出]
    E0[计时 elapsed_time] --> E1[nnia.fitness(B, local_search=False)\n重新评估合并种群]
    E1 --> E2[非支配排序 + 拥挤距离选择\n得到 final (size=Pop_size)]
    E2 --> E3[plot_pareto_fronts_3d\n绘制前沿]
    E3 --> E4[normalize_and_weighted_sum_3\n选 min_index]
    E4 --> E5[Decode(final[min_index])\n得到基准甘特/目标]
    E5 --> E6[ReDecode(final[min_index])\n得到考虑故障的甘特/目标]
    E6 --> E7[Gantt_optimized 绘制甘特图]
    E7 --> E8[write_last_generation_population_3obj_row\n写入 Excel]
    E8 --> E9[write_time(run_times, tag)\n写耗时统计]
  end

  S2 --> E0
