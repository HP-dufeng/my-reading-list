# Builder

有时候对象的创建过程并不是那么简单：
- 根据业务规则，对一些输入参数作有效的验证
- 需要一些代码来改进性能：从缓存中读一些对象
- 创建对象的时候，幂等性？，线程安全？ 等问题
- 对象或许有很多构造参数，有些参数或许是可选的？，参数的顺序？

builder 模式根据你提供的参数，从而创建需要的对象

```
type ReservationBuilder interface {
    Vertical(string) ReservationBuilder
    ReservationDate(string) ReservationBuilder
    Build() Reservation
}

type reservationBuilder struct {
    vertical string
    rdate string
}

func (r *reservationBuilder) Vertocal(v string) ResvervationBuilder {
    r.vertical = v
    return r
}

func (r *reservationBuilder) ReservationDate(date string) ResvervationBuilder {
    r.rdate = date
    return r
}

func (r *reservationBuilder) Build(v string) Reservation {
    var builtReservation Reservation

    switch r.vertical {
        case "flight":
            builtReservation = FlightReservationImpl{r.rdate}
        case "hotel":
            builtReservation = HotelReservationImpl{r.rdate}
    }

    return builtReservation
}

func NewReservationBuilder() ReservationBuilder {
    return &reservationBuilder{}
}

```