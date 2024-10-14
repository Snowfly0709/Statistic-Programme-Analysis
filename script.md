protected void doSolveBackward(CFG<Node> cfg, DataflowResult<Node, Fact> result) {
        // TODO - finish me
        if(!cfg.getIR().getMethod().getName().equals("<init>")) {
            Node exit = cfg.getExit();
            boolean isSame = true;
            // 找到return语句
            while (true) {
                //倒序遍历
                for (Node pred : cfg.getPredsOf(exit)) {
                    while (!pred.equals(cfg.getEntry())) {
                        //meet
                        Fact out = result.getOutFact(pred);
                        //找到所有的后继节点
                        for (Node succ : cfg.getSuccsOf(pred)) {
                            //将所有后继节点的inFact合并
                            System.out.println(result.getInFact(succ));
                            analysis.meetInto(result.getInFact(succ), out);
                        }
                        //transfer
                        Fact in = result.getInFact(pred);
                        //将outFact传递给inFact
                        isSame = analysis.transferNode(pred, in, out) && isSame;
                        pred = cfg.getPredsOf(pred).iterator().next();
                    }
                }
                exit = cfg.getPredsOf(exit).iterator().next();
                if(exit.equals(cfg.getEntry())){
                    //检验这一轮和上一轮IN是否一致
                    if(isSame){
                        break;
                    }
                    else {
                        isSame = true;
                        exit = cfg.getExit();
                    }
                }
            }
        }
    }