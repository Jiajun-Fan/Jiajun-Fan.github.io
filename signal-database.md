# Signal database

## 简介

Signal database根据用户design产生，其目的是为后续的应用（例如simulator，debugger，synthesizer等）提供统一易用的信号接口。它主要有以下特点：

1.  基于folded数据结构，但能提供完整的flatten功能。

    其基本思路是对于任意的flatten signal / scope等，有全局唯一的表示，但并不存在与之对应的数据结构来储存其信息，而是在需要的时候从对应的folded数据结构上获取。
2.  多应用的统一接口。

    Signal database通常在static elaboration之后由unified frontend dump，对于同一design，产生的signal database必然相同。因此有助于在不同工具之间的传递信号相关的信息，避免繁琐且容易出错的通过名字进行的匹配。
3.  Signal database不可修改。

    Signal database是用于表示原始design，因此不可修改。Signal database 可以用于：

    1. 对于Simulator将优化后的信号映射回原始design。
    2. Signal database可以用于waveform创建scope和variable，也可以用于将waveform映射回debug database。
    3. 可以以用于综合器将在综合产生的netlist中标注原始design的RTL信号。

## 基本思路

对于任意flatten signal，可以用两个部分来唯一表示：

1. 全局唯一的flatten scope
2. 在scope下唯一的signal name

因此容易想到，用DFS ID (depth first search ID)来表示flatten scope，用一个Name ID来表示信号名，这样就可以很容易的构建出一个全局唯一的flatten ID。

## DFS / FlatScope

DFS ID的关键在于，DFS ID并不需要真正通过遍历来分配和保存，而是可以通过计算得到。另一方面也可以计算快速从folded数据结构中获取信息。也就是通过计算在folded数据结构上实现flatten的功能。下面具体介绍其实现方式：

1. Folded 数据结构包括两种基本类型
   1. Scope 包括module, program, named block等（具体与Verilog对于scope的定义相同）。
   2. Scope instance。也就是module / program 等的instance，对于top module, package, named scope等，不存在显式的instance，但这里假设存在一个和scope同名的隐式instance。
2. Folded Scope中与DFS相关的数据仅有一个：weight，用于表示当前scope的子树下，所有scope的总数（包含当前scope）。
3. Folded Scope instance中与DFS相关的数据也仅一个：offset，也就是当前instance表示的scope的DFS，与其parent scope的DFS的差值。Offset有一个特点，同一个folded scope instance的不同flatten scope的offset都是相同的，并不因为其DFS的值不同而产生的offset。
4. 获取scope weight和instance offset也很简单：
   1. 对于design中的所有scope做bottom-up的遍历，注意每个folded scope仅需遍历一次。
   2. 对于每个scope时，遍历scope下的每个scope instance。由于是bottom-up遍历，此时scope instance对应scope的weight是已知的。
   3. 对于每个Scope instance：offset = Sum(weight of brothers) + 1
   4. 对于每个Scope：weight = Sum(weight of child instance) + 1

这样就可以完成Folded数据结构的构建。

另一方面，为了快速将DFS关联到Folded数据结构，还需要一个大小为最大DFS+1数组来保存保存Folded Scope instance ID。在下标为DFS处，填入该DFS对应的Folded Scope instance ID。

这样对于任意DFS，可以立即从此数组中获取对应的Folded Scope instance ID，从而找到对应的Folded Scope instance 和与之对应的Folded Scope。

* 获取parent。对于任意DFS，如前所述获取到对应的Folded Scope instance，并从上面得到instance offset，将DFS减去instance offset，即可得到其parent。
* 获取指定的child scope。获取DFS对应的Folded Scope instance，并进一步得到Folded Scope。在Folded Scope上搜索指定名字的child Folded Scope instance，并得到其instance offset。将DFS加上instance offset，即可获得指定的child scope。

```cpp
class Scope {
 public:
    unsigned getWeight() const;
    const Instance& getChild(StrID childName) const;
};

class Instance {
 public:
     static const Instance& getInstance(unsigned instID);
     
     StrID getName() const;
     unsigned getOffset() const;
     const Scope& getScope() const;
};

class FlatScope {
 public:
     const Instance& getInstance() {
          return Instance::getInstance(g_InstID[m_DFS]);
     }
     
     StrID getName() const {
          return getInstance().getName();
     }
     
     std::string getPath(char delimiter) const {
          // do getName() & getParent() till m_DFS == 0
     }
     
     FlatScope getParent() const {
          return FlatScope(m_DFS - getInstance().getOffset());
     }
     
     FlatScope getChild(StrID name) const {
          const Scope& scope = getInstance().getScope();
          const Instance& child = scope.getChild(name);
          return FlatScope(m_DFS + child.getOffset());
     }
     
      
 private:
     unsigned                         m_DFS;
     static std::vector<unsigned>     g_InstID;
};
```

## FlatSignal

如前所述flattened signal由表示FlatScope和信号名的StrID组成，其数据结构大致如下：

```cpp
class FlatSignal {
 public:
     StrID getName() const { return m_Name; }
     std::string getPath(char delimiter) const {
         std::stringstream ss;
         ss << m_Scope.getPath() << delimiter << getName().c_str();
         return ss.str();
     }
     FlatScope getScope() const { return m_Scope; }
     uint64_t getUniqueID() const { return m_ID; }
     
 private:
    union {
        uint64_t                 m_ID;
        struct {
            FlatScope            m_Scope;
            StrID                m_Name;
        };
    };
};

FlatSignal FlatScope::getSignal(StrID name) const {
    return FlatSignal(*this, name);
}
```

通过FlatSignal::getUniqueID()可以得到一个Flattened signal的全局唯一的ID，但在一些情况下，我们希望ID不仅唯一。

* 多数波形格式要求variable的UID是连续的。
* 在需要关联FlatSignal和其它信息时，如果ID不连续，通常需要一个map或者unordered\_map。如果ID是连续的，那么直接使用vector即可，无疑可以改善内存用量和性能。

为了满足这样的需求，需要对Folded Scope数据结构做一些扩展。按照signal声明的先后顺序，将其保存在m\_Signals中。此外通常还需要一个map来完成信号名到m\_Signals中下标的映射(Scope::getSignalID)。

```cpp
class Scope {
 public:
    unsigned getNumSignals() const {
        return m_Signals.size();
    }
    
    unsigned getSignalID(StrID name) const {
        // get the subscript in m_Signals of given name
    } 
    
    StrID getSignalName(unsigned id) const {
        return m_Signals[id];
    }

  private:
    std::vector<StrID>                m_Signals;
};
```

如果scope按照DFS顺序，scope内部signal按照Scope::m\_Signals中的顺序，对所有Flattened signal进行排序，这样就可以得到一个连续的Contiouous ID (ContID)。

为了能够快速从DFS得到ContID，需要一个全局的vector用于保存每个FlatScope中出现的的最大ContID+1，这个值可以在构建DFS时，对每个Scope::getNumSignals()进行累加即可。并且可以通过此ContID得到当前FlatScope中任意的FlatSignal的ContID。

反过来也可以通过ContID，在FlatScope::g\_ContIDs中用两分法查找ContID所在的DFS，得到FlatScope。

```cpp
class FlatScope {
 public:
    uint64_t getContID(StrID name) const {
        const Scope& scope = getInstance().getScope();
        unsigned sigID = scope.getSignalID(name);
        return g_ContIDs[m_DFS] - scope.getNumSignals() + sigID;
    }
    
    static FlatScope getScopeOfContID(uint64_t contID) {
        // 在g_ContIDs中两分法查找
    }
    
    static unsigned getMaxScope() {
        return g_InstIDs.size();
    }
    
 private:
    static std::vector<uint64_t>    g_ContIDs;
};

class FlatSignal {
 public:
     uint64_t getContID() const {
         return m_Scope.getContID(m_Name);
     }
};
```

## Flat API的特点

在整个signal database中，Flatten base的数据结构仅仅是FlatScope::g\_InstIDs和FlatScope::g\_ContIDs两项，其大小为design最大DFS。

其余FlatScope，FlatSignal等实际仅仅是一个integer，它们在使用时被构建，并不持续占用内存。同样因为这个特点，FlatScope / FlatSignal等在传递和保存的过程中应当使用object，而不是指针或者引用。

用Integer表示的数据结构还有一个优点，就是可以通过计算得到，例如深度优先遍历Scope tree。

```cpp
for (unsigned i = 1; i < FlatScope::getMaxScope(); ++i) {
    printf("Visit scope %s\n", FlatScope(i).getPath('.').c_str());
}
```

这样的数据结构也很容易并行

```cpp
parallel_for(range<unsigned>(1, FlatScope::getMaxScope()), [](unsigned i){
    printf("Parallel visit scope %s\n", FlatScope(i).getPath('.').c_str());
});
```
