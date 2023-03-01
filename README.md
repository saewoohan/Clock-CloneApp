# Clock-CloneApp 📱 

아이폰의 기본 시계 앱을 swift를 이용하여 cloneApp을 만들었다. mp3파일이 깨지는 문제가 생겨 소리 기능을 제외한 모든 부분을 똑같이 구현하려고 노력했다. 😅

## 1. World Clock  🕰️

2개의 TableViewController를 사용하여 구현하였다.
가장 어려웠던 부분은, 추가하는 도시의 시간을 어떻게 다음 ViewController에서 이전의 ViewController로 가져올까였다. 방법은 protocol을 사용하여 delegate를 설정해 주면 되었다.

----

#### 전달 받는 ViewController
    
    //protocol 설정
    protocol SendDelegate: AnyObject {
    func send(name: String)
    }
    
    //프로토콜 내부에 있는 send함수.
    func send(name: String) {
        array.append(name)
        UserDefaults.standard.set(array, forKey: "Array")
        tableView.reloadData()
    }
    

    override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
        if segue.identifier == "segueIdentifier" {
            let nextVC = segue.destination as! WorldAddTableViewController
            //다음 VC의 delegate를 설정해줘야지 가능.
            nextVC.delegate = self
        }
    }
    
----

#### 전달 해 주는 ViewController


    weak var delegate: SendDelegate?
  
    //cell을 선택 했을 때, delegate에 있는 send함수를 실행시키고 ViewController를 닫음.
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        self.delegate?.send(name: array[indexPath.row])
        self.dismiss(animated: true)
    }
  
----

또한, 슬라이딩으로 cell을 삭제하는 것과 편집 버튼 클릭 시 cell 편집 모드를 구현했다.

    @IBAction func deleteButton(_ sender: UIButton) {
         if self.tableView.isEditing {
                sender.setTitle("편집", for: .normal)
                self.tableView.setEditing(false, animated: true)        
            } else {
                sender.setTitle("완료", for: .normal)
                self.tableView.setEditing(true, animated: true)       
            }   
        DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 0.3){
            self.tableView.reloadData()
        }
    }
    
    //editingStyle에 따른 실행 코드
    override func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
        if editingStyle == .delete {
            array.remove(at: indexPath.row)
            UserDefaults.standard.set(array, forKey: "Array")
            tableView.deleteRows(at: [indexPath], with: .fade)
            
        } else if editingStyle == .insert {
            
        }
    }
    
    //각 index의 editingStyle을 설정.
    
    override func tableView(_ tableView: UITableView, editingStyleForRowAt indexPath: IndexPath) -> UITableViewCell.EditingStyle {
        return .delete
    }

    // Row Editable true
    override func tableView(_ tableView: UITableView, canEditRowAt indexPath: IndexPath) -> Bool {
        return true
    }
    
    // Move Row Instance Method
    override func tableView(_ tableView: UITableView, moveRowAt sourceIndexPath: IndexPath, to destinationIndexPath: IndexPath) {
        let moveCell = self.array[sourceIndexPath.row]
        self.array.remove(at: sourceIndexPath.row)
        self.array.insert(moveCell, at: destinationIndexPath.row)
        UserDefaults.standard.set(array, forKey: "Array")
    }
    
----

## 2. Alarm ⏰ 

가장 시간을 많이 쓴 기능이었다.. 알람을 추가하고 그 데이터들은 World Clock에서 구현했던 것처럼 B->A 방식으로 하면 되겠다는 생각이 들었다. 하지만 한가지 걸리는 점이 Alarm기능에선 Cell을 터치 했을 시 편집 화면으로 넘어가야 한다는 점이었다.
고민을 많이 한 끝에 protocol 방식을 그래도 채택하되, table이 reload될 때, 각 cell을 delegate처리 해주면 기능이 완벽하게 구현될 것이라고 생각했다. 

### cell을 클릭했을 때, 다음 VC로 넘어가는 상황에서의 B->A 데이터 전달 방식
----

    //프로토콜 추가
    protocol Alarm: AnyObject {
      func send(label: String, h: String, m: String, isON: Bool, text: String)
      func delete()
      func toggle(dayLabel: String, timeLabel: String, isON: Bool)
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        guard let cell = (tableView.dequeueReusableCell(withIdentifier: "AlarmCell", for: indexPath) as? AlarmTableViewCell)
        else{
            fatalError()
        }
        cell.delegate = self
        cell.timeLabel.text = arr[indexPath.row]
        cell.dayLabel.text = label[indexPath.row]
        cell.toggleSwitch.isOn = isON[indexPath.row]

        return cell
    }

----

추가적으로 알람이 울려야 할지 안올려야 할지를 비교하기 위해서 Timer를 사용하여 1초마다 가지고 있는 알람데이터들을 비교해 주었다.

    //1초마다 #selector 함수를 실행시키게 함
    Timer.scheduledTimer(timeInterval: 1.0, target: self, selector: #selector(updateTime), userInfo: nil, repeats: true)
    
    @objc func updateTime()
    {
        let date = NSDate()
        let formatter = DateFormatter()
        formatter.dateFormat = "HH:mm"
        let timeCurrent = formatter.string(from: date as Date)
        formatter.dateFormat = "E"
        let day = formatter.string(from: date as Date)
        if arr.count != 0 {
            for t in 0...arr.count-1 {
                if arr[t] == timeCurrent && (label[t].contains(day)) && isON[t] == true {
                    //play Sound
                }
            }
        }
    }

----


## 3. StopWatch ⏱

가장 구현이 간단하고 시간이 짧게 걸린 기능이었다. DispatchSourceTimer를 사용하여 일시정지 기능을 구현하였다.
또한, 배열과 tableView를 추가로 만들어줘서 랩 기능도 따로 구현하였다.


----

#### 시간 계산하는 코드

    func makeTimer() {
        if timer == nil {
            timer = DispatchSource.makeTimerSource(flags: [], queue: DispatchQueue.main)
            timer?.schedule(deadline: .now(), repeating: 0.001)
            timer?.setEventHandler(handler: {
                self.timeUp()
            })
        }
    }
    
    func timeUp(){
        let timeInterval = Date().timeIntervalSince(startTime)
        var minute = Double((Int)(fmod(timeInterval/60,60))) + time[0]
        var second = Double((Int)(fmod(timeInterval,60))) + time[1]
        var milliSecond = Double((Int)((timeInterval - floor(timeInterval))*1000)) + time[2]
        if minute > 59 {
            minute = minute - 60
        }
        if second > 59 {
            second = second - 60
            minute += 1
        }
        if milliSecond > 999 {
            milliSecond -= 1000
            second += 1
        }
        
        
        miliLabel.text = String(format: "%03.0f", milliSecond)
        secondLabel.text = String(format: "%02.0f", second)
        minuteLabel.text = String(format: "%02.0f", minute)
        
    }
    
-----

## 4. Timer ⏲

stopwatch와 마찬가지로 일시정지 기능을 구현하기 위해 DispatchSourceTimer를 사용해 주었고, 받은 시, 분, 초를 모두 초로 변환하여 1초마다 1초 씩 빼주어 구현하였다. (ex) 1시간 1분 1초 -> timeLeft = 3661)

----

#### 시간 계산 코드

    func makeTimer() {
        if timer == nil {
            timer = DispatchSource.makeTimerSource(flags: [], queue: DispatchQueue.main)
            timer?.schedule(deadline: .now(), repeating: 1.0)
            timer?.setEventHandler(handler: {
                self.timerMethod()
            })
        }
    }
    
    @objc func timerMethod() {
        var temp = timeLeft
        
        var temp_h = 0
        var temp_m = 0
        var temp_s = 0
        
        while temp >= 3600 {
            temp -= 3600
            temp_h += 1
        }
        while temp >= 60 {
            temp -= 60
            temp_m += 1
        }
        temp_s = temp
        timeLabel.text = String(format: "%02.0f", Double(temp_h)) + ":" + String(format: "%02.0f", Double(temp_m)) + "." + String(format: "%02.0f", Double(temp_s))
        
        timeLeft -= 1
        if timeLeft <= 0 {
            timeLeft = 0
            h = 0
            sec = 0
            min = 0
            isStart = false
            stop = false
            pickerView.isHidden = false
            timeLabel.isHidden = true
            startLabel.text = "시작"
            timer?.suspend()
            TimerViewController.sound()
        }
    }
    
    func calcul() {
        timeLeft += sec
        timeLeft += min * 60
        timeLeft += h * 3600
        print(timeLeft)
    }


#### 실행영상 

<img src = "https://user-images.githubusercontent.com/112225610/222139285-16ff7588-3dc6-4329-8156-643c4a713626.gif" width = "30%" height = "50%">
----

## 5. 후기

하루에 2-3시간 씩 약 10일정도가 걸렸다. Udemy강의를 들은 후 처음으로 혼자 프로젝트를 하나 진행해 보았는데, 코딩할 당시에는 정말 어렵게 느껴졌던 부분들이 프로젝트가 끝나고 다시 리뷰해 보니, 
한 기능당 200줄 내외의 짧은 코드밖에 안된다는 것이 자괴감에 빠지게 했다.. 첫 프로젝트라 코드 자체도 정리가 잘 안되있고, 특히 디자인 모델은 MVC를 채택한다고 채택을 해봤는데, 제대로 디자인 모델에 맞게
구현 했는지도 의문이다. 하지만 처음 프로젝트를 진행 할 때는 이 기능을 어떻게 구현하지 막막했지만, 한번 구현해 보니 별거 아닌 기능들이었다. 앞으로 프로젝트를 진행할 때, 비슷한 기능들을 구현해야 할 때가 온다면
훨씬 쉽게 할 수 있을거 같다.





